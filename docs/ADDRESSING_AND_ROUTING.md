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
| `80.80.80.0/24` | Provider | HQ PE-CE link + provider core links |
| `90.90.90.0/30` | Provider | DC PE-CE link |
| `70.70.70.0/30` | Provider | Branch PE-CE link |
| `192.51.100.0/24` | Provider | PE and P router loopbacks |
| `10.255.255.0/30` | Overlay | GRE/IPsec tunnel, HQ to DC |

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

## 3. HQ Site

**AS 65100 · `172.16.0.0/16` · OSPF process 100, area 0**

Collapsed core. Two Layer 3 switches hold every SVI and act as the gateway for all VLANs. Two access switches connect to both cores over LACP Port-Channels and operate purely at Layer 2.

### VLANs

| VLAN | Name | Subnet | Gateway (HSRP VIP) | HQ-Core-SW1 | HQ-Core-SW2 |
|---|---|---|---|---|---|
| 10 | Users | `172.16.10.0/24` | `172.16.10.1` | `.2` | `.3` |
| 20 | Voice | `172.16.20.0/24` | `172.16.20.1` | `.2` | `.3` |
| 30 | Mgmt | `172.16.30.0/24` | `172.16.30.1` | `.2` | `.3` |
| 40 | Servers | `172.16.40.0/24` | `172.16.40.1` | `.2` | `.3` |

### Port-Channels (LACP)

| Po | Side A | Ports | Side B | Ports |
|---|---|---|---|---|
| `Po1` | HQ-Core-SW1 | e0/1, e1/2 | HQ-Core-SW2 | e0/1, e1/2 |
| `Po11` | HQ-Core-SW1 | e0/2, e0/3 | HQ-Access-SW1 | e0/2, e0/3 |
| `Po12` | HQ-Core-SW1 | e1/0, e1/1 | HQ-Access-SW2 | e1/0, e1/1 |
| `Po21` | HQ-Core-SW2 | e1/0, e1/1 | HQ-Access-SW1 | e0/0, e0/1 |
| `Po22` | HQ-Core-SW2 | e0/2, e0/3 | HQ-Access-SW2 | e0/2, e0/3 |

All trunks: 802.1Q, native VLAN 999, allowed VLANs 10,20,30,40.

### Routed Links

| Link | Subnet | HQ-Edge-Router | Core |
|---|---|---|---|
| Edge to HQ-Core-SW1 | `172.16.1.0/30` | `.1` (e0/2) | `.2` |
| Edge to HQ-Core-SW2 | `172.16.1.4/30` | `.5` (e0/1) | `.6` |

### Router IDs

| Device | OSPF Router-ID |
|---|---|
| HQ-Edge-Router | `172.16.255.1` |
| HQ-Core-SW1 | `172.16.255.2` |
| HQ-Core-SW2 | `172.16.255.3` |

### First-Hop Redundancy and Layer 2

**HSRP** — HQ-Core-SW1 is Active on all four VLANs (priority 110, preempt). HQ-Core-SW2 is Standby.

**MST** — HQ-Core-SW1 is root. Instances are load-balanced so both access uplinks carry traffic:

```
Po11 -> VLANs 10, 20, 40 forwarding
Po21 -> VLAN 30 forwarding
```

STP root and HSRP Active are aligned on the same switch so traffic does not hairpin across the core peer-link.

### Routing

OSPF process 100, area 0, between the cores and HQ-Edge-Router.

HQ-Edge-Router redistributes OSPF into BGP toward PE1:

```
router bgp 65100
 address-family ipv4
  redistribute ospf 100 match internal external 1 external 2
```

The `match internal external 1 external 2` clause is required. The cores inject VLAN subnets into OSPF via `redistribute connected`, so they arrive as **O E2** external routes. The default `redistribute ospf` behaviour is `match internal`, which silently skips them.

---

## 4. DC Site

**AS 65200 · `10.0.0.0/16` · OSPF process 1, area 0**

Collapsed core. Two Layer 3 switches hold every SVI and act as the gateway for all server VLANs. Two access switches connect to both cores over LACP Port-Channels and operate purely at Layer 2.

### VLANs

| VLAN | Subnet | Gateway (HSRP VIP) | DC-Core-SW1 | DC-Core-SW2 |
|---|---|---|---|---|
| 10 | `10.0.10.0/24` | `10.0.10.1` | `.2` | `.3` |
| 20 | `10.0.20.0/24` | `10.0.20.1` | `.2` | `.3` |
| 30 | `10.0.30.0/24` | `10.0.30.1` | `.2` | `.3` |
| 40 | `10.0.40.0/24` | `10.0.40.1` | `.2` | `.3` |

### Port-Channels (LACP)

| Po | Core | Ports | Access Switch | Ports |
|---|---|---|---|---|
| `Po1` | DC-Core-SW1 | e0/2, e1/0 | DC-Access-SW1 | e0/0, e0/1 |
| `Po2` | DC-Core-SW1 | e1/1, e1/2 | DC-Access-SW2 | e1/0, e1/1 |
| `Po1` | DC-Core-SW2 | e1/1, e1/2 | DC-Access-SW1 | e1/0, e1/1 |
| `Po2` | DC-Core-SW2 | e0/2, e1/0 | DC-Access-SW2 | e0/0, e0/1 |

All bundles are 802.1Q trunks carrying VLANs 10, 20, 30 and 40.

### Routed WAN Uplinks

| Link | Subnet | DC-WAN-RTR | Core |
|---|---|---|---|
| WAN-RTR to DC-Core-SW1 | `10.0.254.0/30` | `.1` (e0/1) | `.2` (e0/0) |
| WAN-RTR to DC-Core-SW2 | `10.0.254.4/30` | `.5` (e0/2) | `.6` (e0/0) |

These are the only routed interfaces inside the DC. Everything below the cores is Layer 2.

### Loopbacks

| Device | Address |
|---|---|
| DC-Core-SW1 | `10.0.255.1` |
| DC-Core-SW2 | `10.0.255.2` |
| DC-WAN-RTR | `10.0.255.8` |

The access switches also carry loopbacks (`10.0.255.11`, `10.0.255.12`). These are vestigial and serve no function in a Layer 2 device.

### First-Hop Redundancy and Layer 2

**HSRP** — DC-Core-SW1 is Active on all four VLANs (priority 110, preempt). DC-Core-SW2 is Standby.

```
Interface  Grp  Pri P State   Active  Standby     Virtual IP
Vl10        10  110 P Active  local   10.0.10.3   10.0.10.1
Vl20        20  110 P Active  local   10.0.20.3   10.0.20.1
Vl30        30  110 P Active  local   10.0.30.3   10.0.30.1
Vl40        40  110 P Active  local   10.0.40.3   10.0.40.1
```

**Spanning tree** — Rapid-PVST. Each access switch is dual-homed, so STP blocks one uplink per VLAN.

**No core peer-link.** Unlike HQ, the two cores are not directly connected. They reach each other through the access switches over the stretched VLANs, and through DC-WAN-RTR over the routed uplinks. This works, but a direct trunk between them would be the conventional build.

### Routing

OSPF process 1, area 0, between the two cores and DC-WAN-RTR over the `10.0.254.x` links. The cores redistribute their connected VLAN subnets into OSPF.

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
  redistribute ospf 1
```

A discard route anchors the aggregate and acts as a safety net for unrouted `10.0.x.x` traffic:

```
ip route 10.0.0.0 255.255.0.0 Null0 250
```

New subnets added inside the DC are covered automatically by the `/16` and require no changes at any other site.

---

## 5. Branch 1

**AS 65300 · `192.168.0.0/16` · Static routing**

Router-on-a-stick. BR1-SW1 is a pure Layer 2 switch; all inter-VLAN routing happens on BR1-Router via 802.1Q sub-interfaces, and DHCP is served by the router.

### VLANs

| VLAN | Name | Subnet | Gateway | Sub-interface |
|---|---|---|---|---|
| 10 | Users | `192.168.10.0/24` | `.1` | `Ethernet0/1.10` |
| 30 | Mgmt | `192.168.30.0/24` | `.1` | `Ethernet0/1.30` |
| 999 | Native | — | — | `Ethernet0/1.999` (no IP) |

### WAN

| Link | Subnet | PE1 | BR1-Router |
|---|---|---|---|
| PE1 to BR1-Router | `70.70.70.0/30` | `.1` (e0/2) | `.2` (e0/0) |

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

This mirrors how providers actually serve small branches. A site with a single exit has nothing to gain from a dynamic protocol.

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
| PE1 to P-Router | `80.80.80.4/30` | global |
| P-Router to PE-2 | `80.80.80.8/30` | global |
| PE1 to HQ-Edge-Router | `80.80.80.0/30` | VRF |
| PE-2 to DC-WAN-RTR | `90.90.90.0/30` | VRF |
| PE1 to BR1-Router | `70.70.70.0/30` | VRF |

### VRF Definition

```
ip vrf Enterprise_VRF
 rd 65000:1
 route-target export 65000:1
 route-target import 65000:1
```

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

This overlay is the primary path between HQ and DC, not a backup. The MPLS core provides only the underlay reachability it rides on. Branch 1 has no tunnel and depends entirely on the L3VPN.

---

## 8. Routing Protocol Summary

| Scope | Protocol | Details |
|---|---|---|
| HQ internal | OSPF 100, area 0 | Cores to HQ-Edge-Router |
| DC internal | OSPF 1, area 0 | Cores to DC-WAN-RTR over `10.0.254.x` |
| Provider core | OSPF + LDP | PE1 to P-Router to PE-2, loopback reachability |
| HQ to PE1 | eBGP 65100 to 65000 | `redistribute ospf 100 match internal external 1 external 2` |
| DC to PE-2 | eBGP 65200 to 65000 | `redistribute ospf 1` + `aggregate-address summary-only` |
| Branch to PE1 | Static | Default out, statics inbound |
| PE1 to PE-2 | MP-BGP VPNv4 | Loopback peering, RT 65000:1 |
| HQ to DC | eBGP over GRE/IPsec | 65100 to 65200 on Tunnel0 |

---

## 9. Site Comparison

| | HQ | DC | Branch 1 |
|---|---|---|---|
| Pattern | Collapsed Core | Collapsed Core | Router-on-a-Stick |
| L3 devices | Two cores | Two cores | One router |
| Access uplinks | LACP Port-Channel | LACP Port-Channel | Single trunk |
| Loop prevention | MST | Rapid-PVST | None needed |
| Gateway | HSRP VIP | HSRP VIP | Sub-interface |
| Core peer-link | Yes (`Po1`) | No | — |
| Redundancy | HSRP + Po + STP | HSRP + Po + STP | None |
