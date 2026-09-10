# Site Designs

Three customer sites and a service provider core. This document explains what each is, how it is built, and why.

The two enterprise sites share a pattern. The branch uses a deliberately minimal one. The provider core is a different world entirely.

---

## Overview

| | HQ | DC | Branch 1 | Provider |
|---|---|---|---|---|
| Pattern | Collapsed Core | Collapsed Core | Router-on-a-Stick | MPLS L3VPN |
| L3 devices | Two cores | Two cores | One router | PE1, P, PE-2 |
| Access uplinks | LACP Port-Channel | LACP Port-Channel | Single trunk | — |
| Loop prevention | MST | Rapid-PVST | None needed | — |
| Gateway | HSRP VIP | HSRP VIP | Sub-interface | — |
| Core peer-link | Yes (`Po1`) | **No** | — | — |
| Redundancy | HSRP + Po + STP | HSRP + Po + STP | **None** | Single P router |

---

## The Collapsed Core Pattern

Both enterprise sites use it, so it is worth stating once.

Distribution and core collapse into a single pair of Layer 3 switches. Access switches connect to both, VLANs stretch between them, and Layer 2 redundancy is handled by STP plus a first-hop redundancy protocol.

It is the standard campus build, and it is standard because campus and server networks stretch VLANs. A host in VLAN 10 may sit on either access switch, so VLAN 10 must exist on both.

```
        Edge Router
       /            \
  Core-SW1 ====== Core-SW2      (peer-link, where present)
     |    \        /    |
     |     \      /     |       LACP Port-Channels
     |      \    /      |
  Access-SW1     Access-SW2
```

**Three things make it work:**

**LACP Port-Channels** bundle two physical links into one logical link. A single member failing does not take the path down.

**HSRP** presents a virtual IP as the gateway. When the active core fails, the VIP moves rather than disappearing, and hosts notice nothing.

**Spanning tree** prevents the loops that dual-homing necessarily creates, blocking the redundant path until it is needed.

---

## HQ Site

**AS 65100 · `172.16.0.0/16`**

### Port-Channels

| Po | Side A | Ports | Side B | Ports |
|---|---|---|---|---|
| `Po1` | HQ-Core-SW1 | e0/1, e1/2 | HQ-Core-SW2 | e0/1, e1/2 |
| `Po11` | HQ-Core-SW1 | e0/2, e0/3 | HQ-Access-SW1 | e0/2, e0/3 |
| `Po12` | HQ-Core-SW1 | e1/0, e1/1 | HQ-Access-SW2 | e1/0, e1/1 |
| `Po21` | HQ-Core-SW2 | e1/0, e1/1 | HQ-Access-SW1 | e0/0, e0/1 |
| `Po22` | HQ-Core-SW2 | e0/2, e0/3 | HQ-Access-SW2 | e0/2, e0/3 |

All LACP `mode active`. Trunks are 802.1Q with native VLAN 999 and an explicit allowed list.

```
interface range Ethernet0/2 - 3
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,30,40
 channel-protocol lacp
 channel-group 11 mode active
```

### Spanning tree — MST with load balancing

HQ-Core-SW1 is root. Instances are split so both access uplinks carry traffic instead of one sitting idle:

```
Po11 -> VLANs 10, 20, 40 forwarding
Po21 -> VLAN 30 forwarding
```

### The alignment that matters

**STP root and HSRP Active both sit on HQ-Core-SW1.** If the root were SW1 while HSRP Active were SW2, every outbound flow would cross the core peer-link on its way out — traffic zigzagging for no reason.

A small detail with a large effect, and the one most often missed.

### Known limitation

A two-member Port-Channel doubles its STP path cost when one member fails, which can trigger a root-port change and a reconvergence. Where that matters, pin the cost:

```
interface Port-channel11
 spanning-tree cost 1000000
```

---

## DC Site

**AS 65200 · `10.0.0.0/16`**

> **Naming note.** Devices are named `DC-Spine-*` and `DC-Leaf-*` for historical reasons. The implemented design is a collapsed core — the cores hold every SVI and the access switches are pure Layer 2.

Structurally the same as HQ, with three differences worth documenting.

### Difference 1 — Rapid-PVST instead of MST

```
spanning-tree mode rapid-pvst
```

Rapid-PVST runs one spanning tree instance per VLAN. Simpler to reason about, but it does not scale the way MST does — MST maps many VLANs onto a handful of instances, which matters once VLAN counts grow.

With four VLANs the difference is academic. At two hundred it would not be.

### Difference 2 — No core peer-link

HQ has `Po1` joining its two cores directly. The DC does not.

The two cores still reach each other, by two indirect routes: through the access switches over the stretched VLANs, and through DC-WAN-RTR over the routed uplinks. HSRP hellos travel the first of those paths.

It works. A direct trunk between the cores would be the conventional build, and would give HSRP a dedicated path that does not depend on an access switch staying up.

### Difference 3 — Routed uplinks to the WAN router

| Link | Subnet |
|---|---|
| DC-WAN-RTR to DC-Spine-1 | `10.0.254.0/30` |
| DC-WAN-RTR to DC-Spine-2 | `10.0.254.4/30` |

These are the only routed interfaces inside the DC. Everything below the cores is Layer 2.

### Port-Channels

| Po | Core | Ports | Access Switch | Ports |
|---|---|---|---|---|
| `Po1` | DC-Spine-1 | e0/2, e1/0 | DC-Leaf-1 | e0/0, e0/1 |
| `Po2` | DC-Spine-1 | e1/1, e1/2 | DC-Leaf-2 | e1/0, e1/1 |
| `Po1` | DC-Spine-2 | e1/1, e1/2 | DC-Leaf-1 | e1/0, e1/1 |
| `Po2` | DC-Spine-2 | e0/2, e1/0 | DC-Leaf-2 | e0/0, e0/1 |

### HSRP

DC-Spine-1 is Active on all four VLANs, priority 110 with preempt. DC-Spine-2 is Standby.

```
Interface  Grp  Pri P State   Active  Standby     Virtual IP
Vl10        10  110 P Active  local   10.0.10.3   10.0.10.1
Vl20        20  110 P Active  local   10.0.20.3   10.0.20.1
Vl30        30  110 P Active  local   10.0.30.3   10.0.30.1
Vl40        40  110 P Active  local   10.0.40.3   10.0.40.1
```

---

## Branch 1

**AS 65300 · `192.168.0.0/16`**

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

In a twenty-person branch this is irrelevant. In a campus it would be a bottleneck, and that is precisely why HQ and the DC are built differently.

### No redundancy, on purpose

One router, one link, no FHRP. A small branch does not justify the cost of duplication, and this decision is made in the field every day.

There is no routing protocol either. A single default route points at the provider:

```
ip route 0.0.0.0 0.0.0.0 70.70.70.1
```

A site with one exit has nothing to gain from a dynamic protocol.

---

## Provider Core

**AS 65000 · OSPF + LDP · VRF `Enterprise_VRF`**

This is the part of the lab that is genuinely a different design domain, and the reason the project covers more ground than a multi-site campus build.

```
       PE1 -------- P-Router -------- PE-2
        |                              |
   HQ + Branch                        DC
```

### Separation by VRF

All three customer sites live in one VRF:

```
ip vrf Enterprise_VRF
 rd 65000:1
 route-target export 65000:1
 route-target import 65000:1
```

The **route distinguisher** makes overlapping customer prefixes unique inside the provider's tables. The **route targets** control which VRF a route lands in on the far side. Neither exists on the customer routers — the VRF is invisible to them.

### MP-BGP VPNv4

PE routers exchange customer routes over a single iBGP session between loopbacks:

```
address-family vpnv4
 neighbor 192.51.100.2 activate
 neighbor 192.51.100.2 send-community both
 neighbor 192.51.100.2 next-hop-self
```

`send-community both` is what carries the route targets. Without it the session establishes, prefixes cross, and the far PE discards every one of them because it cannot tell which VRF they belong to.

### Three PE-CE arrangements, one lab

This is the interesting part:

| Site | Protocol | Why |
|---|---|---|
| HQ | eBGP, AS 65100 | Many subnets, redistributes from OSPF |
| DC | eBGP, AS 65200 | Summarises with `aggregate-address` |
| Branch | Static routes | One exit, nothing to compute |

All three coexist in the same VRF. A real provider offers exactly this menu, and customers pick per site according to what they can maintain.

### Per-site ASNs

Using one customer ASN everywhere would mean each site rejecting routes that carry its own ASN in the AS-path, and the PEs would need `as-override` to rewrite it. Distinct ASNs sidestep that.

Both patterns appear in the field. Per-site is simpler; one shared ASN with `as-override` is more common in large deployments where managing a table of ASNs is its own burden.

---

## Overlay — GRE over IPsec

Tunnel0 connects HQ-Edge-Router and DC-WAN-RTR directly, riding on the MPLS transport, with eBGP running across it.

**It is the primary path between HQ and the DC, not a backup.** The MPLS core provides only the underlay reachability it needs.

Branch 1 has no tunnel and depends entirely on the L3VPN — which is how a long-standing failure in the provider core was eventually discovered. See [`TROUBLESHOOTING.md`](TROUBLESHOOTING.md).

---

## Failure Coverage

| Failure | HQ | DC | Branch |
|---|---|---|---|
| Single link | Port-Channel absorbs it | Port-Channel absorbs it | — |
| Core switch | HSRP + STP | HSRP + STP | — |
| Access switch | Not covered | Not covered | Not covered |
| WAN | Tunnel + MPLS | Tunnel + MPLS | Not covered |

Both enterprise sites survive a core failure. Neither survives the loss of the access switch a host is plugged into, because each host has a single connection. Covering that requires dual-homed hosts plus MLAG — hardware capability this lab does not have, and a cost real deployments frequently decline.

---

## Why the Branch Is Different

Applying the collapsed core design to the branch would mean four switches and an FHRP for twenty users — expense and complexity with no return.

Applying the branch design to HQ would put every flow through one router on one cable.

The two enterprise sites share a pattern because they share a problem: many hosts, stretched VLANs, and no tolerance for a single device failure. The branch has none of those, and is built accordingly.

**Design follows scale.** That is the whole point.
