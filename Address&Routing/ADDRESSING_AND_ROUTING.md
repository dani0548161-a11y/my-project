Addressing and Routing

Complete IP addressing plan and routing design for the Enterprise Network lab (EVE-NG).

Three customer sites connected over a simulated MPLS L3VPN provider core, plus a GRE/IPsec overlay between HQ and the DC.

1. Address Space Allocation
Block	Owner	Purpose
172.16.0.0/16	HQ Site	Campus VLANs and internal links
10.0.0.0/16	DC Site	Server VLANs, fabric links, loopbacks
192.168.0.0/16	Branch 1	User and management VLANs
80.80.80.0/24	Provider / HQ WAN	PE-CE link + provider core links
90.90.90.0/30	Provider / DC WAN	PE-CE link
70.70.70.0/30	Provider / Branch WAN	PE-CE link
192.51.100.0/24	Provider	PE and P router loopbacks (RFC 5737 TEST-NET-2)
10.255.255.0/30	Overlay	GRE/IPsec tunnel between HQ and DC

VLAN IDs are reused across sites (10 = Users, 20 = Voice, 30 = Mgmt) for operational consistency. Subnets are unique per site.

2. Autonomous Systems
ASN	Entity	PE-CE Protocol
65000	MPLS Provider	—
65100	HQ Site	eBGP
65200	DC Site	eBGP
65300	Branch 1	Static routes

All ASNs are from the private range (64512–65534).

Per-site ASNs avoid the AS-path loop-prevention problem that would otherwise require as-override on the PEs. Branch 1 uses static routing instead of BGP, matching common practice for small sites.


3. DC Site — Leaf-Spine (Routed Access)

AS 65200 · 10.0.0.0/16 · OSPF process 1, area 0

Design Rules
Spines do not connect to spines
Leaves do not connect to leaves
All leaf-spine links are routed /30 point-to-point — no STP in the fabric

Every leaf has 4 uplinks (2 to each spine), producing 4-way ECMP for any inter-leaf or outbound flow.

Server VLANs
VLAN	Subnet	Gateway
10	10.0.10.0/24	.1
20	10.0.20.0/24	.1
30	10.0.30.0/24	.1

DHCP is served locally by the switch holding each SVI (pool starts at .50).

Fabric Links — 10.0.253.0/24
Spine	Port	Address	Leaf	Port	Address
DC-Spine-1	e0/2	10.0.253.1	DC-Leaf-1	e0/0	10.0.253.2
DC-Spine-1	e1/0	10.0.253.5	DC-Leaf-1	e0/1	10.0.253.6
DC-Spine-1	e1/1	10.0.253.9	DC-Leaf-2	e1/0	10.0.253.10
DC-Spine-1	e1/2	10.0.253.13	DC-Leaf-2	e1/1	10.0.253.14
DC-Spine-2	e1/1	10.0.253.17	DC-Leaf-1	e1/0	10.0.253.18
DC-Spine-2	e1/2	10.0.253.21	DC-Leaf-1	e1/1	10.0.253.22
DC-Spine-2	e0/2	10.0.253.25	DC-Leaf-2	e0/0	10.0.253.26
DC-Spine-2	e1/0	10.0.253.29	DC-Leaf-2	e0/1	10.0.253.30

Every fabric interface carries no switchport and ip ospf network point-to-point — the latter suppresses DR/BDR election and speeds convergence.

WAN Uplinks
Link	Subnet	DC-WAN-RTR	Spine
WAN-RTR ↔ DC-Spine-1	10.0.254.0/30	.1 (e0/1)	.2 (e0/0)
WAN-RTR ↔ DC-Spine-2	10.0.254.4/30	.5 (e0/2)	.6 (e0/0)
Router IDs
Device	OSPF Router-ID
DC-Spine-1	10.0.255.1
DC-Spine-2	10.0.255.2
DC-Leaf-1	10.0.255.11
DC-Leaf-2	10.0.255.12
DC-WAN-RTR	10.0.255.8
Routing

DC-WAN-RTR injects a default route into the fabric:

router ospf 1
 default-information originate

And summarises the entire site outward, advertising only the /16 supernet:

router bgp 65200
 address-family ipv4
  network 10.0.0.0 mask 255.255.0.0
  aggregate-address 10.0.0.0 255.255.0.0 summary-only
  redistribute ospf 1

A discard route anchors the aggregate and acts as a safety net for unrouted 10.0.x.x traffic:

ip route 10.0.0.0 255.255.0.0 Null0 250

Consequence: new subnets added inside the DC are covered automatically by the /16 and require no changes at any other site.

4. Branch 1 — Router-on-a-Stick

AS 65300 · 192.168.0.0/16 · Static routing

VLANs
VLAN	Name	Subnet	Gateway	Sub-interface
10	Users	192.168.10.0/24	.1	Ethernet0/1.10
30	Mgmt	192.168.30.0/24	.1	Ethernet0/1.30
999	Native	—	—	Ethernet0/1.999 (no IP)

BR1-SW1 is a pure Layer 2 switch. All inter-VLAN routing happens on BR1-Router via 802.1Q sub-interfaces, and DHCP is served by the router.

WAN
Link	Subnet	PE1	BR1-Router
PE1 ↔ BR1-Router	70.70.70.0/30	.1 (e0/2)	.2 (e0/0)
Routing

BR1-Router runs no routing protocol. A single default route points at the provider:

ip route 0.0.0.0 0.0.0.0 70.70.70.1

PE1 holds matching statics inside the VRF and advertises them into BGP:

ip route vrf Enterprise_VRF 192.168.10.0 255.255.255.0 70.70.70.2
ip route vrf Enterprise_VRF 192.168.30.0 255.255.255.0 70.70.70.2
!
router bgp 65000
 address-family ipv4 vrf Enterprise_VRF
  network 192.168.10.0 mask 255.255.255.0
  network 192.168.30.0 mask 255.255.255.0

This mirrors how providers actually serve small branches. A site with a single exit has nothing to gain from a dynamic protocol.

5. MPLS Provider Core

AS 65000 · OSPF + LDP · VRF Enterprise_VRF

Device Addressing
Device	Interface	Address	Table
PE1	Loopback0	192.51.100.1	global
PE1	e0/0	80.80.80.5	global (to P-Router)
PE1	e0/1	80.80.80.1	VRF (to HQ-Edge-Router)
PE1	e0/2	70.70.70.1	VRF (to BR1-Router)
P-Router	Loopback0	192.51.100.3	global
P-Router	e0/0	80.80.80.6	global (to PE1)
P-Router	e0/1	80.80.80.9	global (to PE-2)
PE-2	Loopback0	192.51.100.2	global
PE-2	e0/0	80.80.80.10	global (to P-Router)
PE-2	e0/1	90.90.90.1	VRF (to DC-WAN-RTR)
Core Links
Segment	Subnet	Table
PE1 ↔ P-Router	80.80.80.4/30	global
P-Router ↔ PE-2	80.80.80.8/30	global
PE1 ↔ HQ-Edge-Router	80.80.80.0/30	VRF
PE-2 ↔ DC-WAN-RTR	90.90.90.0/30	VRF
PE1 ↔ BR1-Router	70.70.70.0/30	VRF
VRF Definition
ip vrf Enterprise_VRF
 rd 65000:1
 route-target export 65000:1
 route-target import 65000:1
MP-BGP VPNv4

Peering runs between PE loopbacks, carried by the core IGP:

router bgp 65000
 neighbor 192.51.100.2 remote-as 65000
 neighbor 192.51.100.2 update-source Loopback0
 !
 address-family vpnv4
  neighbor 192.51.100.2 activate
  neighbor 192.51.100.2 send-community both
  neighbor 192.51.100.2 next-hop-self

PE1 also originates a default route into the VRF (network 0.0.0.0), so every CE receives 0.0.0.0/0 without local configuration.

VRF-Aware Commands

The VRF exists only on the PEs. CE routers are unaware of it and use ordinary global-table commands.

Global	Inside VRF
ping 1.1.1.1	ping vrf Enterprise_VRF 1.1.1.1
show ip route	show ip route vrf Enterprise_VRF
show ip bgp	show ip bgp vpnv4 vrf Enterprise_VRF

PE loopbacks live in the global table — reach them without the vrf keyword.

6. Overlay — GRE over IPsec

Tunnel0 connects HQ-Edge-Router and DC-WAN-RTR directly, riding on top of the MPLS transport.

Endpoint	Tunnel IP	Tunnel Source	Tunnel Destination
HQ-Edge-Router	10.255.255.1	80.80.80.2 (e0/0)	90.90.90.2
DC-WAN-RTR	10.255.255.2	90.90.90.2 (e0/0)	80.80.80.2
interface Tunnel0
 ip address 10.255.255.x 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination <peer WAN address>
 tunnel protection ipsec profile IPSEC_PROFILE_GRE

eBGP runs across the tunnel between AS 65100 and AS 65200:

neighbor 10.255.255.x remote-as <peer ASN>
neighbor 10.255.255.x update-source Tunnel0

This overlay is the primary path between HQ and DC, not a backup. The MPLS core provides only the underlay reachability it rides on. Branch 1 has no tunnel and depends entirely on the L3VPN.

7. Routing Protocol Summary
Scope	Protocol	Details
HQ internal	OSPF 100, area 0	Cores ↔ HQ-Edge-Router
DC fabric	OSPF 1, area 0	ip ospf network point-to-point, 4-way ECMP
Provider core	OSPF + LDP	PE1 ↔ P-Router ↔ PE-2, loopback reachability
HQ ↔ PE1	eBGP 65100 ↔ 65000	redistribute ospf 100 match internal external 1 external 2
DC ↔ PE-2	eBGP 65200 ↔ 65000	redistribute ospf 1 + aggregate-address summary-only
Branch ↔ PE1	Static	Default out, statics inbound
PE1 ↔ PE-2	MP-BGP VPNv4	Loopback peering, RT 65000:1
HQ ↔ DC	eBGP over GRE/IPsec	65100 ↔ 65200 on Tunnel0
