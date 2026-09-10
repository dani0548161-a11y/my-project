# Site Designs

Three sites, three different design patterns. This document explains what each site looks like and why it is built that way.

The short version: design follows scale. A site with 300 users, a site with racks of servers, and a site with twenty people have genuinely different requirements, and copying one pattern to all three would be wrong in two of them.

---

## Comparison

| | HQ | DC | Branch 1 |
|---|---|---|---|
| Pattern | Collapsed Core | Leaf-Spine (routed access) | Router-on-a-Stick |
| L3 devices | Two cores | Two spines + two leaves | One router |
| Inter-layer links | Port-Channel (LACP) | Routed `/30` point-to-point | Single 802.1Q trunk |
| Traffic distribution | LACP, Layer 2 | **ECMP in OSPF, Layer 3** | None |
| Loop prevention | MST blocks ports | No loop exists, no STP | No loop exists |
| Gateway | HSRP virtual IP | Local SVI on the leaf | Router sub-interface |
| Redundancy | HSRP + Port-Channel + MST | 4-way ECMP | **None** |
| Justified for | Hundreds of users | East-west server traffic | Twenty users |

---

## HQ — Collapsed Core

Distribution and core collapse into a single pair of Layer 3 switches. Access switches connect to both, VLANs stretch between them, and Layer 2 redundancy is managed by STP and a first-hop redundancy protocol.

This is the standard campus pattern, and it is standard because campus networks stretch VLANs. A user in VLAN 10 might sit on either access switch, so VLAN 10 has to exist on both.

```
              HQ-Edge-Router
             /              \
    HQ-Core-SW1 ==== Po1 ==== HQ-Core-SW2
       |    \                  /    |
   Po11|     \Po12      Po21  /     |Po22
       |      \              /      |
   HQ-Access-SW1        HQ-Access-SW2
```

### Port-Channels

Each access switch is dual-homed with a two-link LACP bundle to each core. The cores are joined by a peer-link of their own.

| Po | Side A | Ports | Side B | Ports |
|---|---|---|---|---|
| `Po1` | HQ-Core-SW1 | e0/1, e1/2 | HQ-Core-SW2 | e0/1, e1/2 |
| `Po11` | HQ-Core-SW1 | e0/2, e0/3 | HQ-Access-SW1 | e0/2, e0/3 |
| `Po12` | HQ-Core-SW1 | e1/0, e1/1 | HQ-Access-SW2 | e1/0, e1/1 |
| `Po21` | HQ-Core-SW2 | e1/0, e1/1 | HQ-Access-SW1 | e0/0, e0/1 |
| `Po22` | HQ-Core-SW2 | e0/2, e0/3 | HQ-Access-SW2 | e0/2, e0/3 |

All bundles use LACP `mode active`. Trunks are 802.1Q with native VLAN 999 and an explicit allowed list.

```
interface range Ethernet0/2 - 3
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,30,40
 channel-protocol lacp
 channel-group 11 mode active
```

### First-hop redundancy

HSRP runs on all four VLANs. HQ-Core-SW1 is Active with priority 110 and preempt; HQ-Core-SW2 is Standby.

Hosts receive the virtual IP as their gateway, so a core failure is invisible to them — the VIP moves rather than disappearing.

### Spanning tree

MST, with HQ-Core-SW1 as root. Instances are split so both access uplinks carry traffic instead of one sitting idle:

```
Po11 -> VLANs 10, 20, 40 forwarding
Po21 -> VLAN 30 forwarding
```

**STP root and HSRP Active are deliberately on the same switch.** If the root were SW1 while HSRP Active were SW2, every flow would cross the core peer-link on its way out — traffic would zigzag for no reason. Aligning them is a small detail with a large effect, and it is the one most often missed.

### Known limitation

A two-member Port-Channel doubles its STP path cost when one member fails, which can trigger a root-port change and a reconvergence. Where that matters, pin the cost so a partial failure does not move the topology:

```
interface Port-channel11
 spanning-tree cost 1000000
```

---

## DC — Leaf-Spine, Routed Access

The data centre does not run STP at all. Every leaf-spine link is a routed `/30`, and OSPF distributes traffic across all of them simultaneously.

```
                 DC-WAN-RTR
                /           \
        DC-Spine-1        DC-Spine-2
           /    \            /    \
          /      \          /      \
    DC-Leaf-1 ------------------ DC-Leaf-2
        |                            |
   server VLANs                 server VLANs

   Each leaf: 4 uplinks = 4 ECMP paths
```

### Three design rules

**1. Spines do not connect to spines.** Every leaf already reaches every spine, so a spine-to-spine link adds a path that nothing needs and complicates routing.

**2. Leaves do not connect to leaves.** All traffic goes leaf, spine, leaf — always exactly two hops, from any server to any server. That is what makes latency predictable.

**3. A VLAN lives on exactly one leaf.** No Layer 2 stretching, therefore no STP, no large broadcast domains, and no broadcast storm that can cross the fabric.

The result: **every link forwards traffic at once.** Where STP would block half the topology, OSPF spreads load across four equal-cost paths.

### Why it scales

The reason to put spines in the middle only becomes obvious as leaves are added:

| Leaves | Links, full mesh | Links, via spines |
|---|---|---|
| 2 | 1 | 4 |
| 4 | 6 | 8 |
| 8 | 28 | 16 |
| 20 | **190** | 40 |
| 40 | **780** | 80 |

Full mesh grows quadratically. With spines it is linear — each new leaf needs one link per spine, and adding leaf 21 means touching two switches instead of twenty.

### Interface configuration

```
interface Ethernet0/0
 no switchport
 ip address 10.0.253.2 255.255.255.252
 ip ospf network point-to-point
```

`no switchport` moves the port out of Layer 2 entirely. `ip ospf network point-to-point` suppresses DR/BDR election, which is meaningless on a two-router link and only slows convergence.

### The trade-off

What is lost is Layer 2 adjacency between leaves. That matters for classic VM mobility, for clusters whose heartbeat requires a shared broadcast domain, and for broadcast-based discovery.

Modern data centres solve this with **VXLAN/EVPN** — a Layer 2 overlay on top of the routed underlay, giving both mobility and full link utilisation. That is out of scope here, but the routed underlay built in this lab is exactly the foundation VXLAN runs on.

### Separation without stretched VLANs

Tiers are separated by subnet and policed at the routed hop:

```
ip access-list extended WEB-OUT
 permit tcp 10.0.10.0 0.0.0.255 10.0.30.0 0.0.0.255 eq 3306
 deny   ip  10.0.10.0 0.0.0.255 10.0.30.0 0.0.0.255
 permit ip  any any
!
interface Vlan10
 ip access-group WEB-OUT in
```

This is worth stating plainly: **a VLAN was never a separation mechanism.** It provides a broadcast domain, nothing more. Two hosts inside one VLAN talk with no policy applied at all. Routing between tiers creates a checkpoint where policy can actually be enforced — so a routed design gives you *more* control, not less.

For hard multi-tenancy, VRFs give separate routing tables where an ACL mistake cannot leak traffic.

---

## Branch 1 — Router-on-a-Stick

One router, one Layer 2 switch, one trunk between them. Every VLAN gateway is a sub-interface.

```
PE1 ---- BR1-Router ----trunk---- BR1-SW1 ---- PCs
         sub-interfaces            L2 only
```

```
interface Ethernet0/1.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
!
interface Ethernet0/1.999
 encapsulation dot1Q 999 native
```

The native-VLAN sub-interface carries **no IP address**. It exists to tell the router how to handle untagged frames; omitting it produces confusing behaviour that is hard to trace.

### The hairpin

All traffic between VLANs — even between two hosts in the same building — travels up to the router and back down the same cable. The link carries each packet twice.

In a twenty-person branch this is irrelevant. In a campus it would be a bottleneck, and that is precisely why HQ is built differently.

### No redundancy, on purpose

One router, one link, no FHRP. A small branch does not justify the cost of duplication, and this is a decision made in the field every day. The honest way to document a design is to state what it does not protect against.

---

## Failure Coverage

| Failure | HQ | DC | Branch |
|---|---|---|---|
| Single link | Port-Channel absorbs it | ECMP absorbs it | — |
| Distribution / spine device | HSRP + MST | ECMP | — |
| Access switch / leaf | Not covered | Not covered | Not covered |
| WAN | Tunnel + MPLS | Tunnel + MPLS | Not covered |

Both HQ and the DC survive a core-layer failure. Neither survives the loss of the edge switch a host is plugged into, because each host has a single connection. Covering that requires dual-homed hosts plus MLAG or VXLAN/EVPN — hardware capability this lab does not have, and a cost real deployments frequently decline.

---

## Why Not One Pattern Everywhere

Applying the HQ design to the branch would mean four switches and an FHRP for twenty users — expense and complexity with no return.

Applying the branch design to HQ would put every flow through one router on one cable.

Applying the HQ design to the DC would leave half the links blocked by STP, in the one place where east-west bandwidth matters most.

**Each pattern is correct for its site and wrong for the other two.** That is the point of the lab.
