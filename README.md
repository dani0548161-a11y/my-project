# Enterprise Network Lab — EVE-NG

A three-site enterprise WAN built end to end in EVE-NG: two campus sites behind redundant Layer 3 switching, a small branch office, and a simulated service-provider core carrying all three inside an MPLS L3VPN.

Everything in this repository was configured, broken, diagnosed and verified by hand. The documentation reflects the network as it actually runs, not as it was originally planned.

![Topology](topology/topology.png)

---

## Sites

| Site | Topology | Switching | WAN | Redundancy |
|---|---|---|---|---|
| **HQ** | Collapsed Core | MST + HSRP | eBGP — AS 65100 | Full: dual core, dual uplinks |
| **DC** | Collapsed Core | Rapid-PVST + HSRP | eBGP — AS 65200 | Switching only, single WAN router |
| **Branch 1** | Router-on-a-Stick | Single L2 switch | Static default | None |

Three different levels of redundancy, on purpose. Each site gets the design its scale justifies.

The provider core (**AS 65000**) runs OSPF + LDP with two PE routers and one P router. Customer routes live inside VRF `Enterprise_VRF` and cross the core as MP-BGP VPNv4. The P router holds no customer prefixes at all.

A **GRE over IPsec** tunnel connects HQ and the DC directly on top of the MPLS transport and carries eBGP between AS 65100 and AS 65200.

---

## Technologies

**Switching** — 802.1Q trunking · LACP Port-Channels · MST · Rapid-PVST · HSRP

**Routing** — OSPF (multi-process) · eBGP · MP-BGP VPNv4 · route redistribution · route aggregation

**WAN** — MPLS L3VPN · LDP · VRF-lite on the PEs · GRE over IPsec

**Services** — DHCP on SVIs and router sub-interfaces

---

## Repository Layout

```
README.md
│
├── docs/
│   ├── ADDRESSING_AND_ROUTING.md    IP plan, ASNs, protocols, redistribution
│   ├── SITE_DESIGNS.md              L2/L3 design per site and the reasoning
│   └── TROUBLESHOOTING.md           Nine real faults and how each was found
│
├── configs/
│   ├── hq/                          Edge router, core pair, access pair
│   ├── dc/                          WAN router, core pair, access pair
│   ├── branch/                      BR1-Router, BR1-SW1
│   └── provider/                    PE1, P-Router, PE-2
│
└── topology/
    ├── topology.png                 Annotated diagram
    └── lab.unl                      EVE-NG lab file
```

---

## Documentation

**[docs/ADDRESSING_AND_ROUTING.md](docs/ADDRESSING_AND_ROUTING.md)** — the complete address plan. Which block belongs to which site, every VLAN and subnet, all router-IDs and loopbacks, the ASN allocation and why each site has its own, and every redistribution point with the reason it is configured the way it is.

**[docs/SITE_DESIGNS.md](docs/SITE_DESIGNS.md)** — the design decisions. Why access switches stay Layer 2, why STP root and HSRP Active sit on the same switch, why the DC runs Rapid-PVST where HQ runs MST, why the branch runs no routing protocol at all, and what each site loses when a device fails.

**[docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md)** — nine faults that actually occurred, each with the symptom, the wrong hypothesis, the real cause, and the command that settled it. In seven of the nine, the layer that produced the symptom was not the layer that contained the fault.

---

## Address Summary

| Block | Owner |
|---|---|
| `172.16.0.0/16` | HQ |
| `10.0.0.0/16` | DC — advertised outward as a single aggregate |
| `192.168.0.0/16` | Branch 1 |
| `80.80.80.0/24` | Provider core + HQ PE-CE link |
| `90.90.90.0/30` | DC PE-CE link |
| `70.70.70.0/30` | Branch PE-CE link |
| `192.51.100.0/24` | PE and P loopbacks (RFC 5737) |
| `10.255.255.0/30` | GRE/IPsec overlay |

VLAN IDs repeat across sites for operational consistency (10 = Users, 20 = Voice, 30 = Mgmt, 40 = Servers). Subnets are unique per site.

---

## Verification

The state the lab is expected to be in when healthy:

```
show etherchannel summary            every Po (SU), every member (P)
show standby brief                   Core-SW1 Active, Core-SW2 Standby, all VLANs
show spanning-tree vlan 10           root on Core-SW1
show ip ospf neighbor                all FULL
show mpls ldp neighbor               State: Oper
show ip bgp vpnv4 all summary        PE peering Up with a non-zero prefix count
show ip route vrf Enterprise_VRF     all three sites present
```

End-to-end reachability across all three sites is confirmed working:

```
ping 172.16.10.1        HQ gateway
ping 10.0.10.1          DC gateway
ping 192.168.10.1       Branch gateway
```

---

## Running It

1. Import `topology/lab.unl` into EVE-NG.
2. Devices are Cisco IOL (`i86bi` / L2 image). Adjust node templates if your images differ.
3. Paste the matching file from `configs/` into each device, then `write memory` — IOL nodes lose unsaved configuration on restart.
4. Bring up the provider core first (PE1 → P-Router → PE-2), then the site edge routers, then the switches. OSPF and LDP need the underlay before VPNv4 has anything to peer over.

**Resource note.** Around 20 nodes on a constrained host produces delayed OSPF hellos and `Dead timer expired` messages that look exactly like a routing problem. Power down nodes outside the path you are testing. See Case 9 in the troubleshooting log.

---

## Known Limitations

- **Single-homed WAN at the DC.** `DC-WAN-RTR` is a single point of failure for everything leaving the site.
- **No direct core peer-link in the DC.** The two DC core switches are not directly cabled; their HSRP hellos traverse the access layer. Adding a core-to-core Port-Channel is the first improvement to make.
- **No Internet edge.** This is a private WAN only — NAT, DMZ and Internet breakout are out of scope.
- **No tunnel failure detection.** The eBGP session over Tunnel0 gives partial protection, but a silent IPsec failure with the TCP session intact would black-hole traffic. IP SLA + object tracking is the fix and is documented, not deployed.
