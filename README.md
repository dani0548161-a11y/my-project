# Enterprise Network Lab

A multi-site enterprise WAN built in EVE-NG. Three customer sites connected over a simulated MPLS L3VPN provider core, with a GRE/IPsec overlay between headquarters and the data centre.

Each site is deliberately built on a different design pattern, chosen to match its scale.

![Topology](topology/topology.png)

---

## Sites

| Site | Pattern | ASN | Address Space |
|---|---|---|---|
| **HQ** | Collapsed Core | 65100 | `172.16.0.0/16` |
| **DC** | Leaf-Spine, routed access | 65200 | `10.0.0.0/16` |
| **Branch 1** | Router-on-a-Stick | 65300 | `192.168.0.0/16` |
| **Provider** | MPLS L3VPN | 65000 | `192.51.100.0/24` |

---

## What This Demonstrates

**Campus design** — HSRP, MST with per-instance load balancing, LACP Port-Channels, dual-homed access switches.

**Data centre design** — routed leaf-spine fabric with 4-way ECMP and no spanning tree. Every link forwards traffic.

**Branch design** — single router, 802.1Q sub-interfaces, static routing to the provider.

**Service provider** — MPLS core with OSPF and LDP, VRF-based L3VPN, MP-BGP VPNv4 between PE routers, three different PE-CE arrangements.

**Overlay** — GRE over IPsec carrying eBGP between two customer sites.

---

## Documentation

| Document | Contents |
|---|---|
| [`docs/ADDRESSING_AND_ROUTING.md`](docs/ADDRESSING_AND_ROUTING.md) | Full IP plan, ASNs, routing protocols, VRF and VPNv4 |
| [`docs/SITE_DESIGNS.md`](docs/SITE_DESIGNS.md) | Why each site is built the way it is, and the trade-offs |
| [`docs/TROUBLESHOOTING.md`](docs/TROUBLESHOOTING.md) | Problems encountered, root causes, and what they taught |

---

## Repository Layout

```
.
├── README.md
├── docs/
│   ├── ADDRESSING_AND_ROUTING.md
│   ├── SITE_DESIGNS.md
│   ├── TROUBLESHOOTING.md
│   └── screenshots/
├── configs/
│   ├── hq/
│   ├── dc/
│   ├── branch/
│   └── provider/
└── topology/
    ├── topology.png
    └── ENTERPRISE_NETWORK_project.unl
```

`configs/` holds the running configuration of every device, grouped by site. `topology/` contains the EVE-NG project file, so the lab can be imported and run as-is.

---

## Design Comparison

| | HQ | DC | Branch 1 |
|---|---|---|---|
| Pattern | Collapsed Core | Collapsed Core | Router-on-a-Stick |
| L3 devices | Two cores | Two cores | One router |
| Access uplinks | LACP Port-Channel | LACP Port-Channel | Single trunk |
| Loop prevention | MST | Rapid-PVST | None needed |
| Gateway | HSRP VIP | HSRP VIP | Router sub-interface |
| Core peer-link | Yes (`Po1`) | **No** | — |
| Redundancy | HSRP + Po + STP | HSRP + Po + STP | None |
---

## Built With

- EVE-NG Community Edition
- Cisco IOL / IOU images (L2 and L3)
- VPCS for end hosts

Scope is a private enterprise WAN. Internet edge, NAT and DMZ are not included.

---

## Selected Findings

Three issues from `docs/LAB_NOTES.md` that were worth the time they cost:

**A static route that looked valid and dropped everything.** Its next-hop did not exist, but resolved recursively through a `Null0` discard route — so it installed cleanly and black-holed the traffic in silence.

**A backup path concealing a dead one.** VPNv4 peering between the PEs never came up. Nobody noticed, because HQ and the DC talk over a tunnel that bypasses the MPLS core. Adding a branch with no tunnel exposed it in minutes.

**A one-way virtual link impersonating a routing bug.** Hours of protocol debugging ended at an interface counter frozen at 26 packets while the peer's output counter climbed. Nothing was arriving. Redrawing the cable fixed it in thirty seconds.
