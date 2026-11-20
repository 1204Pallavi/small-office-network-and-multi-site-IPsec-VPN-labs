# Project 1: IP Addressing Plan

All addresses are RFC 1918 (private) or RFC 5737 (documentation ranges).

## VLANs and subnets

| VLAN | Name | Subnet | Gateway (CORE-SW SVI) | DHCP range | Purpose |
|------|------|--------|-----------------------|------------|---------|
| 10 | STAFF | 10.10.10.0/24 | 10.10.10.1 | 10.10.10.21 - .254 | Office users |
| 20 | GUEST | 10.10.20.0/24 | 10.10.20.1 | 10.10.20.21 - .254 | Guest wireless, internet only |
| 30 | SERVERS | 10.10.30.0/24 | 10.10.30.1 | static | DNS + web server |
| 99 | MGMT | 10.10.99.0/24 | 10.10.99.1 | 10.10.99.21 - .254 | Device management |
| 999 | BLACKHOLE | none | none | none | Native VLAN on trunks, unused ports |

## Point-to-point links

| Link | Subnet | Side A | Side B |
|------|--------|--------|--------|
| ISP - R1 (WAN) | 203.0.113.0/29 | ISP G0/0 = .1 | R1 G0/0 = .2 |
| R1 - CORE-SW | 10.10.0.0/30 | R1 G0/1 = .1 | CORE-SW G0/1 = .2 (routed port) |

203.0.113.3 is used as the public (static NAT) address of the web server.

## Device interfaces

| Device | Interface | Address | Notes |
|--------|-----------|---------|-------|
| ISP | G0/0 | 203.0.113.1/29 | |
| ISP | Loopback0 | 198.51.100.1/32 | Simulated Internet host |
| R1 | G0/0 | 203.0.113.2/29 | NAT outside |
| R1 | G0/1 | 10.10.0.1/30 | NAT inside |
| CORE-SW | G0/1 | 10.10.0.2/30 | Routed port to R1 |
| CORE-SW | Vlan10/20/30/99 | .1 in each subnet | Default gateways |
| CORE-SW | Fa0/1 | trunk | To SW-ACCESS Gi0/1 |
| CORE-SW | Fa0/2 | access VLAN 30 | To SERVER-PT |
| SW-ACCESS | Vlan99 | 10.10.99.2/24 | Management, gateway 10.10.99.1 |
| SERVER-PT | NIC | 10.10.30.10/24 | Gateway 10.10.30.1, DNS 10.10.30.10 |

## SW-ACCESS port map

| Ports | VLAN | Use | Security |
|-------|------|-----|----------|
| Gi0/1 | trunk | Uplink to CORE-SW | native VLAN 999 |
| Fa0/1 - 8 | 10 | Staff PCs | Port security (max 2, sticky), PortFast + BPDU Guard |
| Fa0/9 - 12 | 20 | Guest AP | PortFast + BPDU Guard (no port security: an AP bridges many MACs) |
| Fa0/13 | 99 | Management PC | Port security (max 2, sticky), PortFast + BPDU Guard |
| Fa0/14 - 24 | 999 | Unused | Shutdown |

## Routing summary

- OSPF process 1, area 0, between R1 and CORE-SW. SVIs are passive.
- R1 has a static default route to the ISP and advertises it with `default-information originate`.
- ISP needs no routes: it only talks to the directly connected 203.0.113.0/29 and its own loopback.
