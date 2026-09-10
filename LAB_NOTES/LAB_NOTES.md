# Lab Notes

Problems encountered while building this lab, what actually caused them, and what they taught. Written down because every one of them cost hours, and every one of them looked like something it was not.

---

## `show ip route` beats `show run`

Two static routes sat in the DC WAN router's configuration:

```
ip route 10.0.100.0 255.255.255.0 10.0.1.2
ip route 10.0.200.0 255.255.255.0 10.0.1.2
```

The next-hop `10.0.1.2` did not exist anywhere in the network. But a catch-all discard route did:

```
ip route 10.0.0.0 255.255.0.0 Null0 250
```

IOS resolved `10.0.1.2` recursively through that `/16`, found `Null0`, and **installed both statics as valid**. They appeared in `show ip route` looking perfectly healthy — and silently dropped every packet destined for the DC.

The configuration was clean. The routing table was not. `show run` cannot show you this; only following the actual forwarding path can.

**Takeaway:** an installed route is not a working route. Check where it actually resolves.

---

## `network` in BGP does not invent networks

The branch was advertising nothing. The BGP session was up, the neighbour was correct, the VRF and route-targets on the PE were right.

The `network` statements were simply missing from `address-family ipv4`.

`network X mask Y` does not create a prefix — it takes a route that **already exists in the local routing table** and injects it into BGP. No matching route, no advertisement, and no error message either.

**Quick check:**
```
show ip bgp
```
The prefix should appear with next-hop `0.0.0.0` and weight `32768`. If it does not, either the statement is missing or the route is absent from the RIB.

---

## `redistribute ospf` skips external routes by default

HQ LAN subnets appeared in the edge router's routing table as `O E2`, but never reached BGP.

The reason is a hidden default:

```
redistribute ospf 100
```
means
```
redistribute ospf 100 match internal
```

`match internal` covers `O` and `O IA` only. Anything external — `O E1`, `O E2` — is skipped silently.

The cores inject their VLAN subnets with `redistribute connected`, so they arrive as **E2**. BGP ignored all of them.

**Fix:**
```
redistribute ospf 100 match internal external 1 external 2
```

---

## A session being up proves nothing about what crosses it

`show ip bgp summary` showing `Up` only means TCP established. Read the `State/PfxRcd` column: an established session with `0` prefixes is an open, empty pipe.

Equally, a GRE tunnel reports `up/up` whenever two conditions hold — the source interface is up, and a route to the destination exists. It does **not** verify anyone is listening. The tunnel here showed `up/up` while the far site was completely unreachable.

**What actually proves a tunnel works:**
```
show crypto ipsec sa | include encaps|decaps
```
Both counters must be moving. Encaps climbing while decaps stays at zero means traffic leaves and nothing returns.

---

## A backup path hides failures

VPNv4 peering between the two PE routers never came up. Nobody noticed for a long time, because HQ and the DC talk over a GRE/IPsec tunnel that bypasses the MPLS core entirely.

Adding Branch 1 exposed it within minutes. The branch has no tunnel and depends on the L3VPN alone, so it was the first thing to fail when the underlay was broken.

**Takeaway:** a site with no alternate path is an excellent diagnostic instrument. Redundancy is valuable, and it also conceals the fact that one of your paths died months ago.

---

## Layer 1 before Layer 3

The longest chase in the project. OSPF would not form between the P router and PE1. Configuration was verified repeatedly: timers matched, area matched, network type matched, no ACLs, no authentication.

Then the counters:

```
P-Router:  packets output  1542 -> 1548     (climbing)
PE1:       packets input     26 ->   26     (frozen)
```

The virtual link in EVE was dead in one direction. Nothing was arriving at PE1 — not OSPF, not CDP, not ARP, not ping. Deleting and redrawing the cable fixed it in thirty seconds.

**Takeaway:** when the configuration checks out repeatedly, stop reading configuration and ask the physical question — *are the packets arriving at all?* An interface counter answers in seconds what protocol debugging cannot answer in hours.

---

## Intermittent usually means resources

Symptoms that came and went: OSPF neighbours flapping with `Dead timer expired`, BGP sessions dropping to `Active`, pings alternating between success and timeout.

The clue was in the numbers — traceroute latencies of 800 ms on a local virtual lab, where single-digit milliseconds are normal.

Around twenty nodes on a constrained host starves CPU. Hello packets arrive late, dead timers expire, adjacencies reset, routes disappear and return.

**Mitigations, in order:**

1. Power down nodes outside the path under test
2. Relax OSPF timers to tolerate the jitter — matching values on **both** ends:
```
interface <X>
 ip ospf dead-interval 120
```
3. Check the host itself with `uptime` and `free -h`

**Takeaway:** in a lab, "works sometimes" is almost never a configuration bug.

---

## `write memory`, every time

IOL nodes lose unsaved configuration on restart. This lab lost work to it more than once, including a switch that reverted to the default hostname and took its entire Port-Channel configuration with it.

---

## EVE-NG Artefacts That Are Not Real Problems

### `%CDP-4-DUPLEX_MISMATCH`

IOL images do not apply `duplex full` consistently across virtual interfaces. Connectivity is unaffected. Once both ends are genuinely correct:

```
no cdp log mismatch duplex
```

### `Shr` in the `show spanning-tree` Type column

IOS derives STP link type from duplex — full gives `P2p`, half gives `Shr`.

This one is **not** cosmetic. On `Shr`, MST and RSTP fall back to legacy timers: 15 seconds listening plus 15 seconds learning. Convergence stretches past 30 seconds, and a failover that should be instant looks like a total outage.

```
interface <X>
 spanning-tree link-type point-to-point
```

Legitimate in a lab — there is no hub on a virtual link.

---

## VRF Commands on a PE

The VRF exists only on provider edge routers. Every command touching customer addresses needs the keyword, and forgetting it produces failures that look like routing problems:

```
ping vrf Enterprise_VRF 80.80.80.2
traceroute vrf Enterprise_VRF 80.80.80.2
show ip route vrf Enterprise_VRF
show ip bgp vpnv4 vrf Enterprise_VRF
```

A plain `ping 80.80.80.2` from a PE consults the **global** table, which has no such route — so a directly connected neighbour appears unreachable while everything is perfectly fine.

The exception: PE loopbacks are provider infrastructure and live in the global table. Reach those **without** the keyword.

CE routers are unaware the VRF exists and use ordinary commands throughout.

---

## Open Items

**Router-ID collision.** PE1 uses OSPF router-ID `10.0.255.1`, which is also DC-Spine-1's loopback. Separate routing tables mean forwarding is unaffected, but the overlap is confusing in output. Renumbering the DC fabric to `10.0.250.x` would resolve it.

**Duplicate VPNv4 neighbours.** Each PE carries a second, non-functional neighbour statement pointing at an OSPF router-ID rather than a routable loopback. Permanently `Idle`; should be removed.

**No tunnel failure detection.** The eBGP session over Tunnel0 gives partial protection, but a silent IPsec failure while TCP survives would black-hole traffic:

```
ip sla 1
 icmp-echo 10.255.255.1 source-interface Tunnel0
 frequency 5
ip sla schedule 1 life forever start-time now
!
track 1 ip sla 1 reachability
```

**Single-homed WAN at the DC.** DC-WAN-RTR is a single point of failure for all DC-to-WAN traffic.

**No Internet edge.** Private WAN only. NAT, DMZ and Internet breakout are out of scope.
