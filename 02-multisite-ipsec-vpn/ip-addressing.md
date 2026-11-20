# Project 2: IP Addressing Plan

All addresses are RFC 1918 (private) or RFC 5737 (documentation ranges used as the "public" side).

## Sites and LANs

| Site | VLAN / LAN | Subnet | Gateway | Notes |
|------|-----------|--------|---------|-------|
| HQ | VLAN 10 ENG | 10.1.10.0/24 | 10.1.10.1 (HSRP VIP) | DS1 = .2, DS2 = .3. DHCP from HQ-R1 (relayed) |
| HQ | VLAN 20 OPS | 10.1.20.0/24 | 10.1.20.1 (HSRP VIP) | DS1 = .2, DS2 = .3. DHCP from HQ-R1 (relayed) |
| Branch 1 | LAN | 10.2.10.0/24 | 10.2.10.1 (BR1-R1 G0/1) | DHCP on BR1-R1 |
| Branch 2 | LAN | 10.3.10.0/24 | 10.3.10.1 (BR2-R1 G0/1) | DHCP on BR2-R1 |

## WAN links (simulated ISP)

| Link | Subnet | ISP side | Site side |
|------|--------|----------|-----------|
| ISP - HQ-R1 | 203.0.113.0/30 | ISP G0/0 = .1 | HQ-R1 G0/0 = .2 |
| ISP - BR1-R1 | 198.51.100.0/30 | ISP G0/1 = .1 | BR1-R1 G0/0 = .2 |
| ISP - BR2-R1 | 192.0.2.0/30 | ISP G0/2 = .1 | BR2-R1 G0/0 = .2 |

## GRE tunnels (protected by IPsec)

| Tunnel | Subnet | HQ end | Branch end |
|--------|--------|--------|------------|
| HQ - BR1 | 172.16.1.0/30 | HQ-R1 Tunnel0 = .1 | BR1-R1 Tunnel0 = .2 |
| HQ - BR2 | 172.16.2.0/30 | HQ-R1 Tunnel1 = .1 | BR2-R1 Tunnel0 = .2 |

## HQ internal links

| Link | Subnet | Side A | Side B |
|------|--------|--------|--------|
| HQ-R1 - DS1 | 10.1.0.0/30 | HQ-R1 G0/1 = .1 | DS1 G0/1 = .2 (routed port) |
| HQ-R1 - DS2 | 10.1.0.4/30 | HQ-R1 G0/2 = .5 | DS2 G0/1 = .6 (routed port) |

Both /30s fall inside 10.1.0.0/29, which is how OSPF picks them up with one `network` statement.

## Layer 2 links at HQ

| Link | Ports | Type |
|------|-------|------|
| DS1 - DS2 | Fa0/23 on both | Trunk (VLANs 10, 20; native 999) |
| DS1 - AS1 | DS1 Fa0/1 - AS1 Fa0/1 | Trunk, Root Guard on DS1 |
| DS2 - AS1 | DS2 Fa0/1 - AS1 Fa0/2 | Trunk, Root Guard on DS2 |

## AS1 port map

| Ports | VLAN | Use | Security |
|-------|------|-----|----------|
| Fa0/1 - 2 | trunk | Uplinks to DS1 / DS2 | DHCP snooping trusted |
| Fa0/3 - 12 | 10 | ENG PCs | Port security (max 2, sticky), PortFast + BPDU Guard, DHCP snooping rate limit |
| Fa0/13 - 22 | 20 | OPS PCs | Same as above |
| Fa0/23 - 24 | 999 | Unused | Shutdown |

## HSRP and STP roles

| VLAN | HSRP active (priority 110, preempt) | HSRP standby (priority 100) | STP root | STP secondary root |
|------|-------------------------------------|-----------------------------|----------|--------------------|
| 10 | DS1 | DS2 | DS1 | DS2 |
| 20 | DS2 | DS1 | DS2 | DS1 |

## IPsec / IKE parameters

| Item | Value |
|------|-------|
| IKE (phase 1) | IKEv1, AES-256, SHA-1, pre-shared key, DH group 2, lifetime 86400 s |
| IPsec (phase 2) | ESP AES-256 + ESP SHA-HMAC, transform set `VPN-TS` |
| Crypto ACL | `permit gre` between the two public tunnel endpoints (mirror image on each side) |
| PSK | `CHANGE_ME` (placeholder, must match on both ends) |

## Routing summary

- OSPF process 1, area 0 everywhere: HQ-R1, DS1, DS2, both tunnels, BR1-R1 and BR2-R1.
- Public interfaces and user-facing LAN interfaces are passive.
- Each router has a static default route to the ISP so the tunnel endpoints can reach each other. The public subnets are never advertised into OSPF (avoids tunnel recursion).
- Branch-to-branch traffic goes via HQ (hub-and-spoke).
