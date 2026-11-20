# Project 1: Small Office Network

Design and implementation of a small office network in Cisco Packet Tracer: segmented VLANs, OSPF routing, DHCP/DNS, NAT, ACL-based access control, port security and SSH-only management.

## Goals

- Segment the office into Staff, Guest, Servers and Management VLANs
- Route between VLANs on a Layer 3 core switch and reach the "Internet" through an edge router
- Provide DHCP and DNS to all users, plus a wireless guest AP
- Share one public IP with PAT and publish the web server with static NAT
- Restrict traffic with ACLs, harden L2 edge ports, allow SSH only from management

## Topology

```
                       [ISP]  lo0 198.51.100.1 ("Internet")
                         | G0/0  203.0.113.1/29
                         |
                         | G0/0  203.0.113.2/29
                       [R1]  edge router: NAT, OSPF default originate, outside ACL
                         | G0/1  10.10.0.1/30
                         |
                         | G0/1  10.10.0.2/30  (routed port)
                     [CORE-SW]  L3 switch: SVIs, DHCP server, OSPF, inter-VLAN ACLs
                     |        |
                Fa0/1 |        | Fa0/2
          (trunk)     |        |  (access VLAN 30)
                     |    [SERVER-PT] 10.10.30.10  DNS + HTTP
               Gi0/1 |
              [SW-ACCESS]  2960: user ports, port security
               |   |   |
            PCs  AP  Mgmt PC
         (VLAN10)(VLAN20)(VLAN99)
```

Devices (Packet Tracer): 2 x 2911 router (R1, ISP), 1 x 3560-24PS (CORE-SW), 1 x 2960-24TT (SW-ACCESS), Server-PT, PCs, an Access Point-PT, a laptop for wireless. Port names may differ slightly depending on your PT version. Adjust the interface names in the configs if so.

See [ip-addressing.md](ip-addressing.md) for the full plan.

## Build steps

1. Place devices and cable them as in the diagram. Use copper straight-through everywhere (Packet Tracer ports auto-MDIX).
2. Paste configs in this order: `ISP.txt`, `R1.txt`, `CORE-SW.txt`, `SW-ACCESS.txt`. Replace `CHANGE_ME` if you want a different lab password.
3. Configure the server statically: IP `10.10.30.10/24`, gateway `10.10.30.1`, DNS `10.10.30.10`. Turn on DNS and HTTP services. Add DNS A records, e.g. `www.office.lab -> 10.10.30.10`.
4. Set PCs to DHCP. Put the staff PCs on SW-ACCESS Fa0/1-8, the AP on Fa0/9, and the management PC on Fa0/13 (the Mgmt PC will get an address from the VLAN 99 pool).
5. On the AP, set SSID `Office-Guest` and connect the laptop wirelessly. It should land in VLAN 20.
6. Run the verification checklist below and save screenshots into `topology/`.

## Verification checklist

| # | Test | Expected result |
|---|------|-----------------|
| 1 | `show ip ospf neighbor` on R1 and CORE-SW | Neighbor in FULL state |
| 2 | `show ip route` on R1 | `O` routes for 10.10.10/20/30/99 |
| 3 | Staff PC gets DHCP address | 10.10.10.x, gateway 10.10.10.1, DNS 10.10.30.10 |
| 4 | Staff PC pings server 10.10.30.10 | Success |
| 5 | Staff PC pings 198.51.100.1 (Internet) | Success, `show ip nat translations` on R1 shows PAT entry |
| 6 | Staff PC opens `http://www.office.lab` | Web page loads |
| 7 | Guest laptop gets address | 10.10.20.x |
| 8 | Guest pings 10.10.10.x or 10.10.30.10 | Blocked (ACL GUEST-IN) |
| 9 | Guest resolves DNS and reaches 198.51.100.1 | Works |
| 10 | Staff PC pings Mgmt VLAN 10.10.99.x | Blocked (ACL STAFF-IN) |
| 11 | SSH from Mgmt PC to 10.10.99.2 / R1 / CORE-SW | Works |
| 12 | SSH from a Staff PC | Refused (vty access-class) |
| 13 | ISP pings 203.0.113.3 and opens HTTP to it | Reaches the web server (static NAT) |
| 14 | ISP tries Telnet/other ports to 203.0.113.3 | Blocked (OUTSIDE-IN) |
| 15 | Plug a second unknown device into a port-secured port, or move a PC | Port err-disabled (`show port-security interface fa0/1`) |

Useful commands: `show vlan brief`, `show interfaces trunk`, `show ip dhcp binding`, `show access-lists`, `show port-security`, `show ip nat statistics`.

## What to write up in your own words

After testing, add a short "Lessons learned" section here: what broke, how you troubleshot it, and what you would change (for example, add a second core switch with HSRP, or move to an ASA firewall). This is what interviewers will ask about.
