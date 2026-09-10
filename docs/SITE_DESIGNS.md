# Site Designs

Layer 2 and Layer 3 design of each site in the lab, and the reasoning behind the choices.

Addressing lives in [ADDRESSING_AND_ROUTING.md](ADDRESSING_AND_ROUTING.md) — this document covers **topology and design decisions**.

---

## 1. Overview

| Site | Topology | Switching | Routing to WAN | Redundancy |
|---|---|---|---|---|
| HQ | Collapsed Core | MST + HSRP | eBGP (AS 65100) | Full — dual core, dual uplinks |
| DC | Collapsed Core | Rapid-PVST + HSRP | eBGP (AS 65200) | Switching only — single WAN router |
| Branch 1 | Router-on-a-Stick | Single L2 switch | Static default | None |

Three different levels of redundancy, on purpose. A real enterprise does not build every site the same way — the design follows the value of what sits behind it.

---

## 2. The Collapsed Core Pattern

Both HQ and the DC use the same building block:

```
        [ Core-SW1 ]=========[ Core-SW2 ]      L3 — SVIs, HSRP, OSPF
             ||   \\         //   ||
             ||    \\       //    ||           LACP Port-Channels
             ||     \\     //     ||
        [ Access-SW1 ]   [ Access-SW2 ]        L2 only — no SVIs, no routing
```

**Core switches (L3)** hold every SVI, run HSRP so each VLAN has one virtual gateway address, and speak OSPF toward the WAN router.

**Access switches (L2)** carry no IP addresses and no routing. They trunk VLANs upward over two Port-Channels — one to each core — and nothing else.

### Why access switches stay Layer 2

Every access switch is dual-homed to both cores. If access switches routed, each VLAN would need an SVI on both of them, and two devices advertising the same subnet split inbound traffic unpredictably.

Keeping them at Layer 2 means:

- The same VLAN can exist on **both** access switches — that is the point, not a problem. A user on Access-SW1 and a user on Access-SW2 share one broadcast domain and one gateway.
- STP blocks the redundant path, HSRP picks the active gateway, and there is exactly one answer to "where is the gateway."
- Access switches become disposable. Replacing one requires no IP plan and no routing config.

### Why the core is collapsed

The classic three-tier model (access → distribution → core) exists to scale a campus across many buildings. A single-building site has nothing to aggregate: distribution and core would carry identical traffic. Collapsing them into one L3 pair removes a hop and a device pair without losing anything.

---

## 3. HQ Site

**AS 65100 · `172.16.0.0/16` · OSPF 100 area 0**

```
                    [ HQ-Edge-Router ]
                    /                \
        172.16.1.0/30              172.16.1.4/30
              /                          \
     [ HQ-Core-SW1 ]===== Po1 =====[ HQ-Core-SW2 ]
        |        \                    /        |
      Po11        Po12            Po21        Po22
        |            \            /            |
  [ HQ-Access-SW1 ]   \        /   [ HQ-Access-SW2 ]
                       (cross-links)
```

### Port-Channels

| Po | Side A | Ports | Side B | Ports | Role |
|---|---|---|---|---|---|
| `Po1` | HQ-Core-SW1 | e0/1, e1/2 | HQ-Core-SW2 | e0/1, e1/2 | Core peer-link |
| `Po11` | HQ-Core-SW1 | e0/2, e0/3 | HQ-Access-SW1 | e0/2, e0/3 | Access uplink |
| `Po12` | HQ-Core-SW1 | e1/0, e1/1 | HQ-Access-SW2 | e1/0, e1/1 | Access uplink |
| `Po21` | HQ-Core-SW2 | e1/0, e1/1 | HQ-Access-SW1 | e0/0, e0/1 | Access uplink |
| `Po22` | HQ-Core-SW2 | e0/2, e0/3 | HQ-Access-SW2 | e0/2, e0/3 | Access uplink |

All Port-Channels use LACP:

```
interface range Ethernet0/2 - 3
 channel-protocol lacp
 channel-group 11 mode active
```

`mode active` on both ends. LACP negotiates, so a miscabled member is refused rather than silently added — unlike `mode on`, which bundles blindly.

All trunks: 802.1Q, native VLAN 999 (unused), allowed VLANs 10,20,30,40.

### The core peer-link

`Po1` between the two cores is what makes the pair behave as one logical gateway:

- Carries HSRP hellos so each switch knows the other's state
- Carries the OSPF adjacency between the cores
- Carries traffic when the STP root and the HSRP Active end up on different switches for a given VLAN

### MST and load balancing

HQ runs MST. HQ-Core-SW1 is root. Instances are mapped so both access uplinks carry traffic rather than one sitting idle:

```
Po11 → VLANs 10, 20, 40 forwarding
Po21 → VLAN 30 forwarding
```

Without this, STP would block one uplink per access switch entirely and half the uplink capacity would be wasted.

### STP root and HSRP Active must match

HQ-Core-SW1 is both the STP root and the HSRP Active router (priority 110, preempt) for all four VLANs.

If they were split — root on SW1, HSRP Active on SW2 — every packet from an access switch would follow the STP tree up to SW1, then cross `Po1` to reach the gateway on SW2. Every flow, permanently, for no reason. Alignment is a one-line design choice that removes a whole class of traffic asymmetry.

### A Port-Channel limitation worth knowing

STP treats a Port-Channel as a single logical port with a single cost. Losing one member of a two-member bundle does **not** raise that cost — the bundle keeps forwarding at half bandwidth and STP never reconverges.

This is normally desirable (no topology change for a single link failure), but it means a degraded bundle can stay in the path while a healthier alternative is blocked. If that matters:

```
port-channel min-links 2
```

The bundle goes down entirely rather than running degraded, and STP re-elects. Not configured in this lab — noted as a deliberate omission.

---

## 4. DC Site

**AS 65200 · `10.0.0.0/16` · OSPF 1 area 0**

```
                    [ DC-WAN-RTR ]
                    /            \
        10.0.254.0/30          10.0.254.4/30
              /                      \
     [ DC-Core-SW1 ]          [ DC-Core-SW2 ]
        |        \              /        |
       Po1       Po2          Po1       Po2
        |           \        /           |
  [ DC-Access-SW1 ]  \      /  [ DC-Access-SW2 ]
                     (cross-links)
```

Same collapsed-core pattern as HQ, with **three deliberate differences**.

### Difference 1 — Rapid-PVST instead of MST

```
spanning-tree mode rapid-pvst
```

MST scales better and is the right answer for a large campus, but it requires every switch to share an identical region name, revision number and VLAN-to-instance map. A four-switch site does not need that overhead, and Rapid-PVST converges just as fast with per-VLAN visibility that is easier to read in `show spanning-tree`.

Running a different mode here is also intentional as a lab: the two sites demonstrate both approaches.

### Difference 2 — No core peer-link

HQ's cores are joined by `Po1`. The DC cores are **not** directly connected.

They still form an OSPF adjacency, but indirectly — via `DC-WAN-RTR`, which both cores connect to over routed `/30` links.

Consequence: HSRP hellos between `DC-Core-SW1` and `DC-Core-SW2` travel over the access-switch Layer 2 path, not a dedicated link. This works, and HSRP is Active/Standby correctly today, but it means the HSRP peering depends on the access layer staying healthy. Adding a direct core-to-core Port-Channel would be the first improvement to make here.

### Difference 3 — Routed WAN uplinks

The cores reach the WAN router over **routed** interfaces, not trunks:

| Link | Subnet | DC-WAN-RTR | Core |
|---|---|---|---|
| WAN-RTR ↔ DC-Core-SW1 | `10.0.254.0/30` | `.1` (e0/1) | `.2` (e0/0) |
| WAN-RTR ↔ DC-Core-SW2 | `10.0.254.4/30` | `.5` (e0/2) | `.6` (e0/0) |

```
interface Ethernet0/0
 no switchport
 ip address 10.0.254.2 255.255.255.252
```

`no switchport` turns a switch port into a router port. No VLAN, no STP, no HSRP — just an OSPF adjacency.

### Port-Channels

| Core | Ports | Access | Ports |
|---|---|---|---|
| DC-Core-SW1 `Po1` | e0/2, e1/0 | DC-Access-SW1 `Po1` | e0/0, e0/1 |
| DC-Core-SW1 `Po2` | e1/1, e1/2 | DC-Access-SW2 `Po2` | e1/0, e1/1 |
| DC-Core-SW2 `Po1` | e1/1, e1/2 | DC-Access-SW1 `Po2` | e1/0, e1/1 |
| DC-Core-SW2 `Po2` | e0/2, e1/0 | DC-Access-SW2 `Po1` | e0/0, e0/1 |

Each access switch is dual-homed — one bundle to each core.

### VLANs and HSRP

Four server VLANs, all terminating on the core pair:

| VLAN | Subnet | VIP (gateway) | DC-Core-SW1 | DC-Core-SW2 | HSRP State |
|---|---|---|---|---|---|
| 10 | `10.0.10.0/24` | `10.0.10.1` | `10.0.10.2` | `10.0.10.3` | SW1 Active |
| 20 | `10.0.20.0/24` | `10.0.20.1` | `10.0.20.2` | `10.0.20.3` | SW1 Active |
| 30 | `10.0.30.0/24` | `10.0.30.1` | `10.0.30.2` | `10.0.30.3` | SW1 Active |
| 40 | `10.0.40.0/24` | `10.0.40.1` | `10.0.40.2` | `10.0.40.3` | SW1 Active |

```
interface Vlan10
 ip address 10.0.10.2 255.255.255.0
 standby 10 ip 10.0.10.1
 standby 10 priority 110
 standby 10 preempt
```

`DC-Core-SW1` is Active on all four VLANs; `DC-Core-SW2` is Standby. Servers point at the `.1` VIP and never notice a failover.

### Access-switch state

`DC-Access-SW1` and `DC-Access-SW2` carry:

- No SVIs
- No OSPF process
- No IP addressing on any Ethernet interface
- A vestigial `Loopback0` (`10.0.255.11` / `10.0.255.12`) left from an earlier routed-access design — unused and safe to remove

### Routing

`DC-WAN-RTR` originates a default route into the site and summarises the site outward:

```
router ospf 1
 default-information originate
!
router bgp 65200
 address-family ipv4
  network 10.0.0.0 mask 255.255.0.0
  aggregate-address 10.0.0.0 255.255.0.0 summary-only
  redistribute ospf 1 match internal external 1 external 2
```

Remote sites see one prefix — `10.0.0.0/16`. Adding a VLAN inside the DC changes nothing anywhere else.

---

## 5. Branch 1

**AS 65300 · `192.168.0.0/16` · Static routing**

```
   [ PE1 ] ---- 70.70.70.0/30 ---- [ BR1-Router ]
                                          |
                                    e0/1 (trunk)
                                          |
                                   [ BR1-SW1 ]  L2 only
                                     /      \
                                 VLAN 10   VLAN 30
```

### Router-on-a-Stick

One physical link between switch and router carries all VLANs as an 802.1Q trunk. The router terminates each VLAN on a sub-interface:

```
interface Ethernet0/1.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
!
interface Ethernet0/1.30
 encapsulation dot1Q 30
 ip address 192.168.30.1 255.255.255.0
!
interface Ethernet0/1.999
 encapsulation dot1Q 999 native
```

### The hairpin

Traffic between VLAN 10 and VLAN 30 travels **up** the trunk to the router and **back down** the same physical link. Both directions share one interface's bandwidth.

At branch scale — a handful of users, most traffic heading to the WAN anyway — this is irrelevant. At campus scale it is unacceptable, which is why HQ and the DC use L3 switches instead.

### No routing protocol

`BR1-Router` has exactly one way out:

```
ip route 0.0.0.0 0.0.0.0 70.70.70.1
```

A dynamic protocol would spend CPU, memory and adjacency state to learn a single fact that never changes. PE1 holds matching statics in the VRF and advertises them into BGP on the branch's behalf.

This is how providers actually serve small sites.

### No redundancy — deliberately

Single router, single switch, single WAN link. Losing any one of them takes the branch offline.

Redundancy costs money. A 15-person branch office does not justify a second router, a second circuit and the operational complexity of managing failover. The design matches the business value, and documenting that reasoning is more useful than pretending every site deserves the same treatment.

---

## 6. Provider Core

**AS 65000 · OSPF + LDP · VRF `Enterprise_VRF`**

```
[ HQ-Edge-Router ] --- [ PE1 ] --- [ P-Router ] --- [ PE-2 ] --- [ DC-WAN-RTR ]
                          |
                    [ BR1-Router ]
```

The provider is a separate administrative domain. It does not learn customer routes into its global table — customer routes live inside a VRF.

### VRF

```
ip vrf Enterprise_VRF
 rd 65000:1
 route-target export 65000:1
 route-target import 65000:1
```

**RD (Route Distinguisher)** makes prefixes unique on the wire. Two customers using `192.168.10.0/24` become `65000:1:192.168.10.0/24` and `65000:2:192.168.10.0/24` — different VPNv4 routes, no collision.

**RT (Route Target)** controls *distribution*. A PE imports a route if its RT matches one the local VRF imports. RD identifies; RT decides who gets it. They carry the same value here, which is conventional and makes them easy to confuse.

### MP-BGP VPNv4

Peering runs between PE loopbacks, reachable through the core IGP:

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

`send-community both` is not optional. Route Targets **are** extended communities — without it the RTs are stripped, the far PE has nothing to match on, and no VPNv4 route is ever imported. The session comes up cleanly and carries nothing.

`P-Router` holds no customer routes at all. It forwards on MPLS labels only, which is the entire point of the architecture — the core scales without ever seeing a customer prefix.

### Three different PE-CE arrangements

| Site | PE | PE-CE protocol | Why |
|---|---|---|---|
| HQ | PE1 | eBGP 65100 | Many prefixes, changes over time |
| DC | PE-2 | eBGP 65200 | Advertises one aggregate |
| Branch 1 | PE1 | Static | Two prefixes, single exit |

PE1 serves two customer sites on separate interfaces, both in the same VRF — which is exactly how a real PE is used.

### Per-site ASNs

Each site has its own ASN rather than one shared customer ASN.

With a shared ASN, a site would receive a route carrying its own ASN in the AS-path and drop it as a loop.
