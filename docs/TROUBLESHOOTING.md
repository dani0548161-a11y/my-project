# Troubleshooting Log

Real faults encountered while building this lab, and how each was actually found.

Every entry follows the same shape: what the symptom looked like, what it appeared to be, what it actually was, and the command that settled it.

The recurring lesson is stated once here because it applied to almost every incident below:

> **The layer that appeared broken was never the layer that was broken.**

---

## Method

Before any of the specific cases, the approach that consistently worked:

1. **Verify the layer below before debugging the layer above.** An OSPF adjacency problem is usually not an OSPF problem.
2. **Read counters, not configuration.** Configuration says what *should* happen. `show interfaces` says what *is* happening.
3. **Prove reachability in the correct routing table.** A ping that fails from the wrong table proves nothing.
4. **Change one thing.** Two changes and a working network teaches nothing about which one mattered.

---

## Case 1 — EtherChannel Up, VLAN 40 Missing

**Symptom.** After the HQ Port-Channels came up, hosts in VLANs 10, 20 and 30 worked. VLAN 40 had no connectivity anywhere.

**What it looked like.** An EtherChannel problem — the bundle had just been built, so the bundle was suspect.

**What it was.** `switchport trunk allowed vlan` had been written with 10,20,30 and VLAN 40 was added to the design afterwards. The trunk was silently dropping it.

```
show interfaces trunk
```

The **Vlans allowed on trunk** column is the answer. A trunk in `trunking` status with the wrong allowed list looks completely healthy in `show etherchannel summary`.

**Fix.**

```
interface Port-channel11
 switchport trunk allowed vlan add 40
```

`add` — not a bare `switchport trunk allowed vlan 40`, which *replaces* the list and removes the other three.

**Lesson.** `show etherchannel summary` proves the bundle formed. It says nothing about what the bundle carries.

---

## Case 2 — `%CDP-4-DUPLEX_MISMATCH` That Was Not a Duplex Mismatch

**Symptom.** Continuous console messages on the HQ and DC switches:

```
%CDP-4-DUPLEX_MISMATCH: duplex mismatch discovered on Ethernet0/2
```

**What it looked like.** A speed/duplex problem serious enough to explain intermittent behaviour elsewhere.

**What it was.** An IOL artefact. `show interfaces status` showed `a-full` on both ends of every link. Virtual interfaces in EVE do not apply `duplex full` consistently, and CDP reports a mismatch that does not exist at the data plane.

**Fix.** Confirm both ends first, then silence it:

```
show interfaces status
!
no cdp log mismatch duplex
```

**What is worth checking anyway.** Duplex does have a real consequence for STP:

```
show spanning-tree interface Ethernet0/2 detail
```

The **Type** column derives from duplex — `P2p` from full, `Shr` from half. On `Shr`, MST and RSTP fall back to legacy 802.1D timers and convergence stretches past 30 seconds. If it ever shows `Shr`:

```
interface Ethernet0/2
 spanning-tree link-type point-to-point
```

**Lesson.** A loud log message is not the same as an impactful one. Verify the claim before acting on it — but check what the underlying property *actually* affects.

---

## Case 3 — Static Routes That Black-Holed Traffic

**Symptom.** Traffic to a DC subnet was dropped silently. No ICMP unreachable, no log, and `show ip route` showed a valid route toward it.

**What it looked like.** A routing protocol failure — the prefix was present, so forwarding should have worked.

**What it was.** Two leftover static routes:

```
ip route 10.0.100.0 255.255.255.0 10.0.1.2
```

The next-hop `10.0.1.2` did not exist anywhere in the topology. Rather than rejecting the static, IOS resolved it **recursively** against the widest matching route in the table:

```
ip route 10.0.0.0 255.255.0.0 Null0 250
```

That discard route existed to anchor the BGP aggregate. The static resolved through it, IOS considered the next-hop reachable, installed the static as valid — and every packet matching it went to `Null0`.

**How it was found.**

```
show ip route 10.0.100.0
```

The output names the recursive next-hop and the interface the route finally resolves to. Seeing `Null0` at the end of a chain that started with a real IP address is the whole diagnosis.

**Fix.** Remove both statics. The prefix was already learned dynamically.

**Lesson.** A discard route used to anchor an aggregate will happily resolve any broken static pointed near it. A route being *present and valid* is not evidence that it forwards.

---

## Case 4 — `redistribute ospf` That Skipped Half the Routes

**Symptom.** `HQ-Edge-Router` was configured to redistribute OSPF into BGP. Some HQ prefixes reached the DC; the VLAN subnets did not. No error, no warning.

**What it looked like.** A BGP advertisement or filtering problem — a route-map dropping prefixes.

**What it was.** The HQ core switches inject their VLAN subnets into OSPF with `redistribute connected`, so those prefixes arrive at the edge router as **O E2** (external type 2), not as internal OSPF routes.

The default behaviour of `redistribute ospf <pid>` is `match internal`. External routes are skipped — silently, with no indication in the running configuration that anything is being filtered.

**How it was found.** Compare what the router *knows* against what it *advertises*:

```
show ip route ospf
show ip bgp neighbors 80.80.80.1 advertised-routes
```

Every prefix missing from the second output carried `O E2` in the first. That correlation is the diagnosis.

**Fix.**

```
router bgp 65100
 address-family ipv4
  redistribute ospf 100 match internal external 1 external 2
```

**Lesson.** An implicit default is harder to find than a wrong value, because nothing in `show running-config` shows it. When redistribution appears partial, check what route *type* the missing prefixes are.

---

## Case 5 — Branch Advertising Nothing

**Symptom.** Branch 1 could reach HQ and the DC. Neither could reach Branch 1.

**What it looked like.** An asymmetric routing or return-path problem in the provider core.

**What it was.** `BR1-Router` had a BGP session that was **Established**, and advertised zero prefixes.

```
show ip bgp neighbors 70.70.70.1 advertised-routes
```
```
Total number of prefixes 0
```

The session was healthy. There were simply no `network` statements telling it what to advertise.

**Fix.**

```
router bgp 65300
 network 192.168.10.0 mask 255.255.255.0
 network 192.168.30.0 mask 255.255.255.0
```

The mask must match the routing table exactly. `network 192.168.10.0` without `mask` assumes classful `/24` here by luck — writing it explicitly removes the guesswork.

**Lesson.** `Established` means the TCP session and the BGP capability exchange succeeded. It says nothing about content. One-way reachability is almost always an advertisement problem on the unreachable side, not a forwarding problem in the middle.

---

## Case 6 — VPNv4 Peering That Had Never Worked

**Symptom.** No customer routes crossing between PE1 and PE-2. Investigated only after end-to-end traffic failed — the sessions had been broken since they were first configured.

**What it looked like.** MPLS label distribution, or a Route Target mismatch.

**What it was.** Each PE carried **two** neighbour statements:

```
show ip bgp vpnv4 all summary
```
```
Neighbor        V   AS  State/PfxRcd
10.0.255.2      4 65000  Idle
192.51.100.2    4 65000  Idle
```

`10.0.255.2` is an **OSPF router-ID**, not an address assigned to any interface. It is not routable, so that session sat permanently in `Idle`. It had been added early on by copying a router-ID out of `show ip ospf neighbor` and mistaking it for a loopback.

The real loopback pair — `192.51.100.1` / `192.51.100.2` — was also `Idle`, for a separate reason covered in Case 8.

**Fix.** Delete the dead pair so the output shows only sessions that are supposed to work:

```
router bgp 65000
 no neighbor 10.0.255.2 remote-as 65000
```

**Also verified while there.** VPNv4 will not carry Route Targets without:

```
address-family vpnv4
 neighbor 192.51.100.2 send-community both
```

RTs *are* extended communities. Without `send-community both` they are stripped, the far PE has nothing to match against, and no route is imported into the VRF — while the session shows `Established` and a prefix count of zero.

**Lesson.** An OSPF router-ID looks exactly like an IP address and is frequently not one. Confirm with `show ip interface brief` before using anything as a peering address.

---

## Case 7 — PE1 Could Not Ping Its Own Neighbour

**Symptom.** From PE1:

```
ping 80.80.80.2
```
```
.....
Success rate is 0 percent (0/5)
```

`80.80.80.2` is `HQ-Edge-Router`, directly connected on `Ethernet0/1`.

**What it looked like.** A dead link, or an interface in the wrong state.

**What it was.** `Ethernet0/1` is in `Enterprise_VRF`. The ping was issued from the **global** table, which has no route to `80.80.80.0/30` at all — because that subnet exists only inside the VRF.

**Fix.**

```
ping vrf Enterprise_VRF 80.80.80.2
```

**The full set.**

| Global table | Inside the VRF |
|---|---|
| `ping 80.80.80.2` | `ping vrf Enterprise_VRF 80.80.80.2` |
| `show ip route` | `show ip route vrf Enterprise_VRF` |
| `show ip bgp` | `show ip bgp vpnv4 vrf Enterprise_VRF` |
| `traceroute 80.80.80.2` | `traceroute vrf Enterprise_VRF 80.80.80.2` |

PE **loopbacks** live in the global table — reach those *without* the `vrf` keyword. The same device answers differently depending on which table the question is asked in.

**Lesson.** On a PE, "can I ping it" is an incomplete question. The complete question is "can I ping it *from which table*."

---

## Case 8 — The One-Way Link

The longest chase in the project.

**Symptom.** OSPF between PE1 and `P-Router` would not reach FULL.

- On `P-Router`: neighbour stuck in `INIT`
- On PE1: `show ip ospf neighbor` completely empty

**What it looked like.** Everything, in order:

- MTU mismatch → checked, identical
- Network type mismatch (broadcast vs point-to-point) → checked, identical
- Authentication → none configured either side
- Area mismatch → same area
- Subnet mask mismatch → same `/30`
- `debug ip ospf adj` → showed hellos being *sent* from PE1, and nothing received on either side matching them

Every configuration comparison came back clean. `INIT` on one side and empty on the other is the classic signature of one-way hello reception, but the configuration gave no reason for it.

**What it was.** The virtual link in EVE-NG was passing traffic in one direction only. Nothing on either router was wrong.

**How it was settled.** Interface counters, sampled twice a few seconds apart:

| Device | Counter | First read | Second read |
|---|---|---|---|
| P-Router `e0/0` | `packets output` | 1542 | 1548 |
| PE1 `e0/0` | `packets input` | 26 | 26 |

```
show interfaces Ethernet0/0 | include packets input|packets output
```

`P-Router` was transmitting. PE1's input counter was frozen. Frames were leaving one side and never arriving at the other — which no routing protocol configuration can cause.

**Fix.** Delete the cable in the EVE topology and draw it again. The adjacency came up immediately.

**Lesson.** This is the single most transferable item in the log. **Read the counters before reading the configuration.** Two `show interfaces` samples take fifteen seconds and would have replaced an hour of protocol comparison. If one side's `packets output` climbs while the other side's `packets input` does not move, the problem is beneath the protocol entirely — and in a virtual lab, redrawing the link is the fix.

---

## Case 9 — Instability That Was Resource Pressure

**Symptom.** Intermittently across the whole topology:

```
%OSPF-5-ADJCHG: Process 1, Nbr 10.0.255.2 on Ethernet0/1 from FULL to DOWN,
Neighbor Down: Dead timer expired
```

Pings dropping randomly. Traceroute hops showing 800 ms inside a lab where every device is on the same physical host.

**What it looked like.** A routing loop, or link instability.

**What it was.** EVE host CPU starvation. Roughly 20 IOL nodes on a constrained host means hello packets are not generated on time. OSPF interprets a late hello exactly as it interprets a dead neighbour.

**How it was identified.** Two signals together:

- The flaps were **not** correlated with any single link or device — they moved around
- 800 ms latency between two nodes on the same hypervisor is physically impossible unless the *scheduler*, not the network, is the delay

**Mitigations.**

Power down every node outside the path under test — the most effective single action.

Then relax the timers on links that still flap, on **both** ends:

```
interface Ethernet0/1
 ip ospf dead-interval 120
```

Mismatched timers prevent adjacency entirely, so this must be symmetric.

**Lesson.** In a virtual lab, "the network is unstable" and "the host is overloaded" produce identical symptoms. Latency that is physically impossible for the topology is the tell.

---

## Command Reference

The commands that actually resolved things, grouped by what they answer.

**Is the physical path working?**
```
show interfaces Ethernet0/0 | include packets input|packets output
show cdp neighbors
show interfaces status
```

**Is Layer 2 correct?**
```
show etherchannel summary
show interfaces trunk
show spanning-tree vlan 10
show spanning-tree interface Ethernet0/2 detail
show standby brief
```

**Is the route real, or does it resolve into a hole?**
```
show ip route 10.0.100.0
show ip cef 10.0.100.0
```

**Is the prefix being advertised?**
```
show ip bgp neighbors <peer> advertised-routes
show ip bgp neighbors <peer> received-routes
show ip route ospf
```

**Is the VRF correct?**
```
show ip route vrf Enterprise_VRF
show ip bgp vpnv4 all summary
show ip bgp vpnv4 vrf Enterprise_VRF
ping vrf Enterprise_VRF <address>
```

**Is MPLS working?**
```
show mpls ldp neighbor
show mpls forwarding-table
```

---

## Summary

| # | Symptom | Suspected | Actual cause |
|---|---|---|---|
| 1 | One VLAN unreachable | EtherChannel | Missing from trunk allowed list |
| 2 | Duplex mismatch logs | Speed/duplex | IOL artefact |
| 3 | Silent packet loss | Routing protocol | Static resolving through `Null0` |
| 4 | Partial redistribution | BGP filtering | `redistribute ospf` default `match internal` |
| 5 | One-way reachability | Provider core | No `network` statements |
| 6 | No VPN routes | MPLS labels / RT | Peering to an OSPF router-ID |
| 7 | Ping to neighbour fails | Dead link | Wrong routing table |
| 8 | OSPF stuck in INIT | Protocol mismatch | One-way virtual link in EVE |
| 9 | Random flapping | Routing loop | Host CPU starvation |

Nine faults. In seven of them, the layer that produced the symptom was not the layer that contained the fault.
