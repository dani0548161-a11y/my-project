# Addressing and Routing

Complete IP addressing plan and routing design for the Enterprise Network lab (EVE-NG).

Three customer sites connected over a simulated MPLS L3VPN provider core, plus a GRE/IPsec overlay between HQ and the DC.

---

## 1. Address Space Allocation

| Block | Owner | Purpose |
|---|---|---|
| `172.16.0.0/16` | HQ Site | Campus VLANs and internal links |
| `10.0.0.0/16` | DC Site | Server VLANs, WAN uplinks, loopbacks |
| `192.168.0.0/16` | Branch 1 | User and management VLANs |
| `80.80.80.0/24` | Provider / HQ WAN | PE-CE link + provider core links |
| `90.90.90.0/30` | Provider / DC WAN | PE-CE link |
| `70.70.70.0/30` | Provider / Branch WAN | PE-CE link |
| `192.51.100.0/24` | Provider | PE and P router loopbacks (RFC 5737 TEST-NET-2) |
| `10.255.255.0/30` | Overlay | GRE/IPsec tunnel between HQ and DC |

VLAN IDs are reused across sites (10 = Users, 20 = Voice, 30 = Mgmt, 40 = Servers) for operational consistency. Subnets are unique per site.

---

## 2. Autonomous Systems

| ASN | Entity | PE-CE Protocol |
|---|---|---|
| `65000` | MPLS Provider | — |
| `65100` | HQ Site | eBGP |
| `65200` | DC Site | eBGP |
| `65300` | Branch 1 | Static routes |

All ASNs are from the private range (`64512–65534`).

Per-site ASNs avoid the AS-path loop-prevention problem that would otherwise require `as-override` on the PEs. Branch 1 uses static routing instead of BGP, matching common practice for small sites.

---

## 3. HQ Site — Collapsed Core

**AS 65100 · `172.16.0.0/16` · OSPF process 100, area 0**

### VLANs

| VLAN | Name | Subnet | Gateway (HSRP VIP) | HQ-Core-SW1 | HQ-Core-SW2 |
|---|---|---|---|---|---|
| 10 | Users | `172.16.10.0/24` | `172.16.10.1` | `.2` | `.3` |
| 20 | Voice | `172.16.20.0/24` | `172.16.20.1` | `.2` | `.3` |
| 30 | Mgmt | `172.16.30.0/24` | `172.16.30.1` | `.2` | `.3` |
| 40 | Servers | `172.16.40.0/24` | `172.16.40.1` | `.2` | `.3` |

### Routed Links

| Link | Subnet | HQ-Edge-Router | Core |
|---|---|---|---|
| Edge ↔ HQ-Core-SW1 | `172.16.1.0/30` | `.1` (e0/2) | `.2` |
| Edge ↔ HQ-Core-SW2 | `172.16.1.4/30` | `.5` (e0/1) | `.6` |

### Router IDs

| Device | OSPF Router-ID |
|---|---|
| HQ-Edge-Router | `172.16.255.1` |
| HQ-Core-SW1 | `172.16.255.2` |
| HQ-Core-SW2 | `172.16.255.3` |

### Port-Channels (LACP `mode active`)

| Po | Side A | Ports | Side B | Ports |
|---|---|---|---|---|
| `Po1` | HQ-Core-SW1 | e0/1, e1/2 | HQ-Core-SW2 | e0/1, e1/2 |
| `Po11` | HQ-Core-SW1 | e0/2, e0/3 | HQ-Access-SW1 | e0/2, e0/3 |
| `Po12` | HQ-Core-SW1 | e1/0, e1/1 | HQ-Access-SW2 | e1/0, e1/1 |
| `Po21` | HQ-Core-SW2 | e1/0, e1/1 | HQ-Access-SW1 | e0/0, e0/1 |
| `Po22` | HQ-Core-SW2 | e0/2, e0/3 | HQ-Access-SW2 | e0/2, e0/3 |

All trunks: 802.1Q, native VLAN 999, allowed VLANs 10,20,30,40.

### First-Hop Redundancy and Layer 2

**HSRP** — HQ-Core-SW1 is Active on all four VLANs (priority 110, preempt enabled). HQ-Core-SW2 is Standby.

**MST** — HQ-Core-SW1 is the root bridge. Instances are load-balanced so both access uplinks carry traffic:

```
Po11 → VLANs 10, 20, 40 forwarding
Po21 → VLAN 30 forwarding
```

STP root and HSRP Active are deliberately aligned on the same switch so traffic does not hairpin across the core peer-link.

### Routing

OSPF process 100, area 0, between the cores and HQ-Edge-Router.

HQ-Edge-Router redistributes OSPF into BGP toward PE1:

```
router bgp 65100
 address-family ipv4
  redistribute ospf 100 match internal external 1 external 2
```

> The `match internal external 1 external 2` clause is required. The cores inject VLAN subnets into OSPF via `redistribute connected`, so they arrive as **O E2** external routes. The default `redistribute ospf` behaviour is `match internal`, which silently skips them.

---

## 4. DC Site

**AS 65200 · `10.0.0.0/16` · OSPF process 1, area 0**

Four switches: `DC-Spine-1` and `DC-Spine-2` carry every SVI and all Layer 3 state; `DC-Leaf-1` and `DC-Leaf-2` are Layer 2 only. `DC-WAN-RTR` connects the site to the provider.

### Server VLANs

| VLAN | Subnet | Gateway (HSRP VIP) | DC-Spine-1 | DC-Spine-2 |
|---|---|---|---|---|
| 10 | `10.0.10.0/24` | `10.0.10.1` | `10.0.10.2` | `10.0.10.3` |
| 20 | `10.0.20.0/24` | `10.0.20.1` | `10.0.20.2` | `10.0.20.3` |
| 30 | `10.0.30.0/24` | `10.0.30.1` | `10.0.30.2` | `10.0.30.3` |
| 40 | `10.0.40.0/24` | `10.0.40.1` | `10.0.40.2` | `10.0.40.3` |

All four SVIs live on the Spine pair. `DC-Leaf-1` and `DC-Leaf-2` hold **no SVIs, no IP addressing and no routing process** — they trunk VLANs upward and nothing else.

### Port-Channels (LACP `mode active`)

| Po | Spine | Ports | Leaf | Ports |
|---|---|---|---|---|
| `Po1` | DC-Spine-1 | e0/2, e1/0 | DC-Leaf-1 (`Po1`) | e0/0, e0/1 |
| `Po2` | DC-Spine-1 | e1/1, e1/2 | DC-Leaf-2 (`Po2`) | e1/0, e1/1 |
| `Po1` | DC-Spine-2 | e1/1, e1/2 | DC-Leaf-1 (`Po2`) | e1/0, e1/1 |
| `Po2` | DC-Spine-2 | e0/2, e1/0 | DC-Leaf-2 (`Po1`) | e0/0, e0/1 |

Each Leaf is dual-homed — one bundle to each Spine.

There is **no direct link between the two Spines**. Unlike HQ, the Spine pair has no peer-link; their OSPF adjacency is formed indirectly through `DC-WAN-RTR`.

### WAN Uplinks (routed)

| Link | Subnet | DC-WAN-RTR | Spine |
|---|---|---|---|
| WAN-RTR ↔ DC-Spine-1 | `10.0.254.0/30` | `.1` (e0/1) | `.2` (e0/0) |
| WAN-RTR ↔ DC-Spine-2 | `10.0.254.4/30` | `.5` (e0/2) | `.6` (e0/0) |

```
interface Ethernet0/0
 no switchport
 ip address 10.0.254.2 255.255.255.252
```

### Loopbacks and Router IDs

| Device | Loopback0 | Note |
|---|---|---|
| DC-Spine-1 | `10.0.255.1` | OSPF router-ID |
| DC-Spine-2 | `10.0.255.2` | OSPF router-ID |
| DC-WAN-RTR | `10.0.255.8` | OSPF router-ID |
| DC-Leaf-1 | `10.0.255.11` | Unused — no routing process on this device |
| DC-Leaf-2 | `10.0.255.12` | Unused — no routing process on this device |

### Spanning Tree and HSRP

```
spanning-tree mode rapid-pvst
```

`DC-Spine-1` is HSRP Active on all four VLANs (priority 110, preempt). `DC-Spine-2` is Standby.

```
interface Vlan10
 ip address 10.0.10.2 255.255.255.0
 standby 10 ip 10.0.10.1
 standby 10 priority 110
 standby 10 preempt
```

### Routing

DC-WAN-RTR injects a default route into the site:

```
router ospf 1
 default-information originate
```

And summarises the entire site outward, advertising only the `/16` supernet:

```
router bgp 65200
 address-family ipv4
  network 10.0.0.0 mask 255.255.0.0
  aggregate-address 10.0.0.0 255.255.0.0 summary-only
  redistribute ospf 1 match internal external 1 external 2
```

A discard route anchors the aggregate and acts as a safety net for unrouted `10.0.x.x` traffic:

```
ip route 10.0.0.0 255.255.0.0 Null0 250
```

**Consequence:** new subnets added inside the DC are covered automatically by the `/16` and require no changes at any other site.

> This discard route is also what made the black-holing static in Case 3 of the troubleshooting log possible. A broken static pointed anywhere inside `10.0.0.0/16` will resolve recursively through it and install as valid.

---

## 5. Branch 1 — Router-on-a-Stick

**AS 65300 · `192.168.0.0/16` · Static routing**

### VLANs

| VLAN | Name | Subnet | Gateway | Sub-interface |
|---|---|---|---|---|
| 10 | Users | `192.168.10.0/24` | `.1` | `Ethernet0/1.10` |
| 30 | Mgmt | `192.168.30.0/24` | `.1` | `Ethernet0/1.30` |
| 999 | Native | — | — | `Ethernet0/1.999` (no IP) |

BR1-SW1 is a pure Layer 2 switch. All inter-VLAN routing happens on BR1-Router via 802.1Q sub-interfaces, and DHCP is served by the router.

### WAN

| Link | Subnet | PE1 | BR1-Router |
|---|---|---|---|
| PE1 ↔ BR1-Router | `70.70.70.0/30` | `.1` (e0/2) | `.2` (e0/0) |

### Routing

BR1-Router runs no routing protocol. A single default route points at the provider:

```
ip route 0.0.0.0 0.0.0.0 70.70.70.1
```

PE1 holds matching statics inside the VRF and advertises them into BGP:

```
ip route vrf Enterprise_VRF 192.168.10.0 255.255.255.0 70.70.70.2
ip route vrf Enterprise_VRF 192.168.30.0 255.255.255.0 70.70.70.2
!
router bgp 65000
 address-family ipv4 vrf Enterprise_VRF
  network 192.168.10.0 mask 255.255.255.0
  network 192.168.30.0 mask 255.255.255.0
```

> This mirrors how providers actually serve small branches. A site with a single exit has nothing to gain from a dynamic protocol.

---

## 6. MPLS Provider Core

**AS 65000 · OSPF + LDP · VRF `Enterprise_VRF`**

### Device Addressing

| Device | Interface | Address | Table |
|---|---|---|---|
| PE1 | Loopback0 | `192.51.100.1` | global |
| PE1 | e0/0 | `80.80.80.5` | global (to P-Router) |
| PE1 | e0/1 | `80.80.80.1` | VRF (to HQ-Edge-Router) |
| PE1 | e0/2 | `70.70.70.1` | VRF (to BR1-Router) |
| P-Router | Loopback0 | `192.51.100.3` | global |
| P-Router | e0/0 | `80.80.80.6` | global (to PE1) |
| P-Router | e0/1 | `80.80.80.9` | global (to PE-2) |
| PE-2 | Loopback0 | `192.51.100.2` | global |
| PE-2 | e0/0 | `80.80.80.10` | global (to P-Router) |
| PE-2 | e0/1 | `90.90.90.1` | VRF (to DC-WAN-RTR) |

### Core Links

| Segment | Subnet | Table |
|---|---|---|
| PE1 ↔ P-Router | `80.80.80.4/30` | global |
| P-Router ↔ PE-2 | `80.80.80.8/30` | global |
| PE1 ↔ HQ-Edge-Router | `80.80.80.0/30` | VRF |
| PE-2 ↔ DC-WAN-RTR | `90.90.90.0/30` | VRF |
| PE1 ↔ BR1-Router | `70.70.70.0/30` | VRF |

### VRF Definition

```
ip vrf Enterprise_VRF
 rd 65000:1
 route-target export 65000:1
 route-target import 65000:1
```

**RD** makes prefixes unique on the wire. **RT** controls which VRFs import them. They carry the same value here, which is conventional.

### MP-BGP VPNv4

Peering runs between PE loopbacks, carried by the core IGP:

```
router bgp 65000
 neighbor 192.51.100.2 remote-as 65000
 neighbor 192.51.100.2 update-source Loopback0
 !
 address-family vpnv4
  neighbor 192.51.100.2 activate
  neighbor 192.51.100.2 send-community both
  neighbor 192.51.100.2 next-hop-self
```

`send-community both` is mandatory — Route Targets are extended communities, and without it they are stripped and no route is ever imported.

PE1 also originates a default route into the VRF (`network 0.0.0.0`), so every CE receives `0.0.0.0/0` without local configuration.

### VRF-Aware Commands

The VRF exists only on the PEs. CE routers are unaware of it and use ordinary global-table commands.

| Global | Inside VRF |
|---|---|
| `ping 1.1.1.1` | `ping vrf Enterprise_VRF 1.1.1.1` |
| `show ip route` | `show ip route vrf Enterprise_VRF` |
| `show ip bgp` | `show ip bgp vpnv4 vrf Enterprise_VRF` |

PE loopbacks live in the **global** table — reach them without the `vrf` keyword.

---

## 7. Overlay — GRE over IPsec

Tunnel0 connects HQ-Edge-Router and DC-WAN-RTR directly, riding on top of the MPLS transport.

| Endpoint | Tunnel IP | Tunnel Source | Tunnel Destination |
|---|---|---|---|
| HQ-Edge-Router | `10.255.255.1` | `80.80.80.2` (e0/0) | `90.90.90.2` |
| DC-WAN-RTR | `10.255.255.2` | `90.90.90.2` (e0/0) | `80.80.80.2` |

```
interface Tunnel0
 ip address 10.255.255.x 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination <peer WAN address>
 tunnel protection ipsec profile IPSEC_PROFILE_GRE
```

eBGP runs across the tunnel between AS 65100 and AS 65200:

```
neighbor 10.255.255.x remote-as <peer ASN>
neighbor 10.255.255.x update-source Tunnel0
```

**This overlay is the primary path between HQ and DC**, not a backup. The MPLS core provides only the underlay reachability it rides on. Branch 1 has no tunnel and depends entirely on the L3VPN.

---

## 8. Routing Protocol Summary

| Scope | Protocol | Details |
|---|---|---|
| HQ internal | OSPF 100, area 0 | Cores ↔ HQ-Edge-Router |
| DC internal | OSPF 1, area 0 | Spines ↔ DC-WAN-RTR over routed `/30` links |
| Provider core | OSPF + LDP | PE1 ↔ P-Router ↔ PE-2, loopback reachability |
| HQ ↔ PE1 | eBGP 65100 ↔ 65000 | `redistribute ospf 100 match internal external 1 external 2` |
| DC ↔ PE-2 | eBGP 65200 ↔ 65000 | `redistribute ospf 1` + `aggregate-address summary-only` |
| Branch ↔ PE1 | Static | Default out, statics inbound |
| PE1 ↔ PE-2 | MP-BGP VPNv4 | Loopback peering, RT 65000:1 |
| HQ ↔ DC | eBGP over GRE/IPsec | 65100 ↔ 65200 on Tunnel0 |

---

## 9. Site Comparison

| | HQ | DC | Branch 1 |
|---|---|---|---|
| L3 boundary | HQ-Core-SW1/2 | DC-Spine-1/2 | BR1-Router sub-interfaces |
| Access layer | L2 only | L2 only | L2 only |
| L3-pair peer-link | `Po1` | None — via DC-WAN-RTR | n/a |
| STP mode | MST | Rapid-PVST | Default |
| FHRP | HSRP | HSRP | None |
| IGP | OSPF 100 | OSPF 1 | None |
| To provider | eBGP 65100 | eBGP 65200 + aggregate | Static default |

---

## 10. Verification

### HQ and DC Layer 2
```
show etherchannel summary      # every Po (SU), every port (P)
show standby brief             # SW1 Active, SW2 Standby, all VLANs
show spanning-tree vlan 10     # SW1 is root
show interfaces trunk          # allowed VLANs 10,20,30,40
```

### DC Layer 3
```
show ip interface brief | include Vlan     # SVIs on Spines only
show ip ospf neighbor                      # Spines ↔ DC-WAN-RTR, FULL
show ip route 0.0.0.0                      # default learned from DC-WAN-RTR
```

### Provider core
```
show ip ospf neighbor
show mpls ldp neighbor                 # State: Oper
show ip bgp vpnv4 all summary          # PE peering Up with prefix count
show ip route vrf Enterprise_VRF
```

### End to end
```
ping 172.16.10.1        # HQ gateway
ping 10.0.10.1          # DC gateway
ping 192.168.10.1       # Branch gateway
```

---

## 11. Design Notes

**Aggregation at the DC edge.** `aggregate-address 10.0.0.0 255.255.0.0 summary-only` means remote sites see one prefix instead of every server subnet. Adding a VLAN inside the DC requires no change anywhere else.

**Per-site ASNs avoid `as-override`.** With one shared customer ASN, a site would reject routes carrying its own ASN in the AS-path, and the PEs would need `as-override` to rewrite it. Distinct ASNs sidestep this entirely.

**VLAN IDs reused, subnets unique.** VLAN 10 means "Users" at every site, which keeps operations consistent. The subnets differ so routing stays unambiguous.

**Native VLAN 999 is intentionally unused.** Standard hardening against VLAN-hopping — untagged frames land in a VLAN that carries no traffic.

**STP root aligned with HSRP Active.** Both sit on the same switch at each site. Misaligning them sends traffic across the core peer-link on every flow.

**Access switches carry no Layer 3.** Each access switch is dual-homed to both L3 switches. If they routed, the same subnet would be advertised from two devices and inbound traffic would split unpredictably. Keeping them at Layer 2 leaves exactly one gateway per VLAN.

---

## 12. Known Issues and Open Items

**Router-ID collision.** PE1 uses OSPF router-ID `10.0.255.1`, which is also DC-Spine-1's loopback. The two live in separate routing tables (provider global vs. customer VRF) so forwarding is unaffected, but the overlap is confusing in troubleshooting output. Renumbering the DC loopbacks to `10.0.250.x` would resolve it.

**Duplicate VPNv4 neighbours.** PE1 and PE-2 each carry a second, non-functional neighbour statement pointing at an OSPF router-ID rather than a routable loopback. These sit permanently in `Idle` and should be removed.

**No peer-link between the Spines.** `DC-Spine-1` and `DC-Spine-2` are not directly cabled. HSRP hellos between them traverse the Leaf layer, so the FHRP peering depends on the access switches staying healthy. A direct Port-Channel between the Spines is the first improvement to make.

**Unused loopbacks on the Leaves.** `10.0.255.11` and `10.0.255.12` are not referenced by any process. They can be removed.

**Tunnel failure detection.** The eBGP session over Tunnel0 provides partial protection, but a silent IPsec failure while the TCP session survives would black-hole traffic. Mitigation:

```
ip sla 1
 icmp-echo 10.255.255.1 source-interface Tunnel0
 frequency 5
ip sla schedule 1 life forever start-time now
!
track 1 ip sla 1 reachability
```

**No Internet edge.** This is a private WAN only. NAT, DMZ and Internet breakout are out of scope.

**Single-homed WAN at the DC.** DC-WAN-RTR is a single point of failure for all DC-to-WAN traffic.

---

## 13. Lab Notes (EVE-NG)

**`write memory` after every change.** IOL nodes lose unsaved configuration on restart.

**`%CDP-4-DUPLEX_MISMATCH`** is an IOL artefact — virtual interfaces do not apply `duplex full` consistently. It does not affect connectivity. Silence with `no cdp log mismatch duplex`.

**`show spanning-tree` Type column.** `P2p` derives from full duplex, `Shr` from half. On `Shr`, MST/RSTP fall back to legacy timers and convergence stretches past 30 seconds. Force it if needed:

```
interface <X>
 spanning-tree link-type point-to-point
```

**Check the counters before the configuration.** A one-way virtual link in EVE presents exactly like a routing bug. If `packets input` on one side stays frozen while the peer's `packets output` climbs, nothing is arriving — delete and redraw the link rather than debugging the protocol.

**Resource pressure looks like instability.** Roughly 20 nodes on a constrained host produces delayed OSPF hellos, `Dead timer expired`, and intermittent packet loss. Power down nodes outside the path under test.
