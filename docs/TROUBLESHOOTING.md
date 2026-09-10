# Troubleshooting

Problems encountered while building this lab, what actually caused them, and how each was identified.

Every one of them looked like something it was not. That is the recurring theme, and the reason this file exists.

---

## Contents

1. [A static route that installed cleanly and dropped everything](#1-a-static-route-that-installed-cleanly-and-dropped-everything)
2. [`network` in BGP does not invent networks](#2-network-in-bgp-does-not-invent-networks)
3. [`redistribute ospf` skips external routes by default](#3-redistribute-ospf-skips-external-routes-by-default)
4. [A session being up proves nothing about what crosses it](#4-a-session-being-up-proves-nothing-about-what-crosses-it)
5. [A backup path hides a dead one](#5-a-backup-path-hides-a-dead-one)
6. [Layer 1 before Layer 3](#6-layer-1-before-layer-3)
7. [Intermittent usually means resources](#7-intermittent-usually-means-resources)
8. [Environment notes](#8-environment-notes)
9. [Open items](#9-open-items)

---

## 1. A static route that installed cleanly and dropped everything

**Symptom** — no connectivity from HQ to the DC server subnets. Configuration reviewed several times and found correct.

**What the routing table showed:**

```
S    10.0.100.0/24 [1/0] via 10.0.1.2
S    10.0.200.0/24 [1/0] via 10.0.1.2
```

Both routes present, both marked valid.

**Cause** — the next-hop `10.0.1.2` did not exist anywhere in the network. But a catch-all discard route did:

```
ip route 10.0.0.0 255.255.0.0 Null0 250
```

IOS resolved `10.0.1.2` recursively through that `/16`, found `Null0`, and installed both statics as valid. They looked perfectly healthy and black-holed every packet in silence.

Worse, they carried AD 1, so they beat the OSPF routes that later arrived with the correct next-hop.

**Fix**

```
no ip route 10.0.100.0 255.255.255.0 10.0.1.2
no ip route 10.0.200.0 255.255.255.0 10.0.1.2
```

**Lesson** — an installed route is not a working route. Follow where the next-hop actually resolves, and remember that a leftover static outranks a correct dynamic route.

---

## 2. `network` in BGP does not invent networks

**Symptom** — the branch site advertised nothing. BGP session up, neighbour correct, VRF and route-targets on the PE verified.

**What proved it:**

```
BR1-Router#show ip bgp neighbors 70.70.70.1 advertised-routes
Total number of prefixes 0
```

And the local table was equally empty of the branch prefixes.

**Cause** — the `network` statements were missing from `address-family ipv4`.

`network X mask Y` does not create a prefix. It takes a route that **already exists in the local routing table** and injects it into BGP. No matching route, no advertisement — and no error message either.

**Fix**

```
router bgp 65300
 address-family ipv4
  network 192.168.10.0 mask 255.255.255.0
  network 192.168.30.0 mask 255.255.255.0
```

**Quick check** — the prefix should appear in `show ip bgp` with next-hop `0.0.0.0` and weight `32768`. If it does not, either the statement is missing or the route is absent from the RIB.

---

## 3. `redistribute ospf` skips external routes by default

**Symptom** — HQ LAN subnets sat in the edge router's routing table but never reached BGP.

**What the table showed:**

```
O E2  172.16.10.0/24 [110/20] via 172.16.1.6, Ethernet0/1
O E2  172.16.20.0/24 [110/20] via 172.16.1.6, Ethernet0/1
O E2  172.16.30.0/24 [110/20] via 172.16.1.6, Ethernet0/1
O E2  172.16.40.0/24 [110/20] via 172.16.1.6, Ethernet0/1
```

Present, valid, and completely absent from `show ip bgp`.

**Cause** — a hidden default. This:

```
redistribute ospf 100
```

means this:

```
redistribute ospf 100 match internal
```

`match internal` covers `O` and `O IA` only. External routes — `O E1`, `O E2` — are skipped silently.

The cores inject their VLAN subnets with `redistribute connected`, so they arrive as **E2**. BGP ignored all four.

**Fix**

```
redistribute ospf 100 match internal external 1 external 2
```

**Lesson** — the `E2` tag was visible the whole time. Reading the route code, not just the prefix, would have found this in minutes.

---

## 4. A session being up proves nothing about what crosses it

Two variations of the same mistake appeared in this lab.

### BGP

```
Neighbor        V      AS  MsgRcvd MsgSent  Up/Down    State/PfxRcd
80.80.80.2      4   65100       45      50  00:36:30              2
```

`Up` means the TCP session established. Nothing more. The `State/PfxRcd` column is the one that matters — two prefixes where six were expected meant the LAN subnets were missing, which is how issue #3 surfaced.

An established session showing `0` is an open, empty pipe.

### GRE tunnel

A GRE interface reports `up/up` whenever two conditions hold: the source interface is up, and a route to the destination exists. It does **not** verify that anyone is listening.

The tunnel here showed `up/up` while the far site was entirely unreachable.

**What actually proves a tunnel works:**

```
show crypto ipsec sa | include encaps|decaps
```

```
#pkts encaps: 136, #pkts encrypt: 136
#pkts decaps: 138, #pkts decrypt: 138
```

Both counters must move. Encaps climbing while decaps stays at zero means traffic leaves and nothing comes back.

---

## 5. A backup path hides a dead one

**Symptom** — the newly added branch could not reach either other site, while HQ and the DC communicated normally.

**What proved it:**

```
PE1:   10.0.255.3     65000   0  0   never   Idle
PE-2:  10.0.255.1     65000   0  0   never   Idle
```

`never` in the Up/Down column. The VPNv4 peering between the two PE routers had not merely failed — it had **never come up at all**. The MPLS L3VPN had never carried a single route between sites.

Nobody noticed, because HQ and the DC talk over a GRE/IPsec tunnel that bypasses the MPLS core entirely.

Branch 1 has no tunnel and depends on the L3VPN alone. It exposed the problem within minutes of being connected.

**Lesson** — a site with no alternate path is an excellent diagnostic instrument. Redundancy is valuable, and it also conceals the fact that one of your paths died months ago. Test each path in isolation, not just end-to-end reachability.

---

## 6. Layer 1 before Layer 3

The longest chase in the project.

**Symptom** — OSPF would not form between the P router and PE1. Configuration verified repeatedly: timers matched, area matched, network type matched, no ACLs, no authentication mismatch. The P router showed the neighbour stuck in `INIT`; PE1's neighbour table was empty.

**What settled it** — the interface counters, sampled thirty seconds apart:

```
P-Router:   packets output   1542 -> 1548     climbing
PE1:        packets input      26 ->   26     frozen
```

Twenty-six packets total, ever. Nothing was arriving at PE1 — not OSPF, not CDP, not ARP, not ping. The virtual link in EVE was dead in one direction.

**Fix** — delete the link in EVE and redraw it. Thirty seconds.

**Lesson** — when the configuration checks out repeatedly, stop reading configuration and ask the physical question: *are the packets arriving at all?* An interface counter answers in seconds what protocol debugging could not answer in hours.

`show cdp neighbors` is the fastest first check — it runs at Layer 2 and does not depend on IP at all.

---

## 7. Intermittent usually means resources

**Symptom** — OSPF adjacencies flapping, BGP sessions dropping to `Active`, pings alternating between success and timeout.

```
%OSPF-5-ADJCHG: Process 100, Nbr 172.16.255.3 from FULL to DOWN, Dead timer expired
%OSPF-5-ADJCHG: Process 100, Nbr 172.16.255.2 from FULL to DOWN, Dead timer expired
```

**The clue was in the latency** — traceroute times of 800 ms on a local virtual lab, where single-digit milliseconds are normal.

**Cause** — around twenty nodes on a constrained host starves CPU. Hello packets arrive late, dead timers expire, adjacencies reset, routes disappear and return.

The knock-on effect matters: when OSPF drops, the routes it carried vanish from the RIB, so `redistribute ospf` has nothing to convert, and the prefixes disappear from BGP too. A single resource problem presents as a routing failure three layers up.

**Mitigations, in order:**

1. Power down nodes outside the path under test
2. Relax OSPF timers to tolerate the jitter — matching values on **both** ends:

```
interface <X>
 ip ospf dead-interval 120
```

3. Check the host itself with `uptime` and `free -h`

**Lesson** — in a lab, "works sometimes" is almost never a configuration bug.

---

## 8. Environment Notes

### `write memory`, every time

IOL nodes lose unsaved configuration on restart. This lab lost work to it more than once, including a switch that reverted to the default hostname `Switch` and took its entire Port-Channel configuration with it.

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

### VRF-aware commands on a PE

The VRF exists only on provider edge routers. Every command touching customer addresses needs the keyword:

```
ping vrf Enterprise_VRF 80.80.80.2
traceroute vrf Enterprise_VRF 80.80.80.2
show ip route vrf Enterprise_VRF
show ip bgp vpnv4 vrf Enterprise_VRF
```

A plain `ping 80.80.80.2` from a PE consults the **global** table, which has no such route — so a directly connected neighbour appears unreachable while everything is fine.

The exception: PE loopbacks are provider infrastructure and live in the global table. Reach those **without** the keyword.

CE routers are unaware the VRF exists and use ordinary commands throughout.

---

## 9. Open Items

**Router-ID collision.** PE1 uses OSPF router-ID `10.0.255.1`, which is also DC-Spine-1's loopback. Separate routing tables mean
