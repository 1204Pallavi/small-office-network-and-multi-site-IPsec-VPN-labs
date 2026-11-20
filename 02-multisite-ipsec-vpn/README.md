# Project 2: Secure Multi-Site Enterprise Network with IPsec VPN & Redundancy

A headquarters and two branch offices connected over a simulated ISP using GRE-over-IPsec tunnels, with OSPF running across the tunnels. The HQ LAN is built without single points of failure at Layer 2 and Layer 3 (RSTP + HSRP) and the access layer is protected against rogue switches, rogue DHCP servers and MAC spoofing.

## Goals

- Connect HQ, Branch 1 and Branch 2 with encrypted site-to-site tunnels (IKEv1 + ESP AES-256/SHA, GRE for routing support)
- Run OSPF over the GRE tunnels so sites learn each other's LANs dynamically
- Remove single points of failure in the HQ LAN: dual distribution switches, HSRP gateways, RSTP
- Load-balance by VLAN: DS1 is HSRP active and STP root for VLAN 10, DS2 for VLAN 20
- Harden the access layer: Root Guard, BPDU Guard, DHCP snooping, sticky port security

## Topology

```
  BR1-PC --- [BR1-SW] --- [BR1-R1]                                  [HQ-R1] --- G0/1 --- [DS1] ---+
  10.2.10.0/24         G0/0 198.51.100.2/30                         G0/0 203.0.113.2/30      |     |
                              |                                          |             G0/2  |   Fa0/23
                              | G0/1 198.51.100.1/30                     | G0/0 .1           |     |
                              +-------------------[ ISP ]----------------+              [DS2] ---+
                              | G0/2 192.0.2.1/30     (no routes needed,                  |  Fa0/1
                              |                        all links connected)               |
                       G0/0 192.0.2.2/30                                         DS1 Fa0/1 and DS2 Fa0/1
  BR2-PC --- [BR2-SW] --- [BR2-R1]                                                   both trunk to [AS1]
  10.3.10.0/24                                                                           |   |
                                                                                     VLAN10  VLAN20 PCs

  Logical view (GRE over IPsec):
      HQ-R1 Tunnel0 172.16.1.1/30 <==== IPsec ====> BR1-R1 Tunnel0 172.16.1.2/30
      HQ-R1 Tunnel1 172.16.2.1/30 <==== IPsec ====> BR2-R1 Tunnel0 172.16.2.2/30
```

Layer 2 at HQ is a triangle: DS1 - DS2 (Fa0/23), AS1 - DS1 (Fa0/1), AS1 - DS2 (Fa0/2 on AS1 to Fa0/1 on DS2). RSTP blocks one link per VLAN so there is no loop.

Devices (Packet Tracer): 4 x 2911 router (HQ-R1, BR1-R1, BR2-R1, ISP), 2 x 3560-24PS (DS1, DS2), 1 x 2960-24TT (AS1), a switch or hub for each branch (no config needed), PCs. Interface names may differ slightly between PT versions.

See [ip-addressing.md](ip-addressing.md) for the full plan.

## Prerequisites

The IPsec commands need the security technology package on the three VPN routers (HQ-R1, BR1-R1, BR2-R1). On each, once:

```
enable
configure terminal
license boot module c2900 technology-package securityk9
end
write memory
reload
```

Check with `show version` (the license section should list securityk9 as "Technology Package License Information"). Do this before pasting the configs.

## Build steps

1. Place and cable the devices as in the diagram.
2. Enable `securityk9` on HQ-R1, BR1-R1 and BR2-R1 (see Prerequisites).
3. Paste configs in this order: `ISP.txt`, then `HQ-R1.txt`, `BR1-R1.txt`, `BR2-R1.txt`, then `DS1.txt`, `DS2.txt`, `AS1.txt`. Replace the `CHANGE_ME` pre-shared key if you like, but it must be identical on both ends of each tunnel.
4. Set all PCs to DHCP. Put HQ PCs on AS1 Fa0/3-12 (VLAN 10) and Fa0/13-22 (VLAN 20). Branch PCs get their addresses from their own router.
5. Generate some traffic (ping across sites) to bring the IKE negotiation up. The first pings may time out while the tunnel builds.
6. Work through the checklists below and save screenshots and `show` output into `topology/`.

## Verification checklist

### VPN and routing

| # | Test | Expected result |
|---|------|-----------------|
| 1 | `show interface tunnel 0` on HQ-R1 and BR1-R1 | up/up |
| 2 | `show crypto isakmp sa` on HQ-R1 | Two entries in state QM_IDLE (BR1 and BR2) |
| 3 | `show crypto ipsec sa` on HQ-R1 | `#pkts encaps` and `#pkts decaps` increase while you ping |
| 4 | `show ip ospf neighbor` on HQ-R1 | FULL with 172.16.1.2, 172.16.2.2, DS1 and DS2 |
| 5 | `show ip route ospf` on BR1-R1 | Routes to 10.1.10.0, 10.1.20.0 and 10.3.10.0 via 172.16.1.1 |
| 6 | HQ PC (VLAN 10) pings BR1 PC | Success |
| 7 | BR1 PC pings BR2 PC | Success (traffic hairpins through HQ) |
| 8 | `tracert` from HQ PC to BR1 PC | Shows the tunnel address 172.16.1.2, not the ISP |
| 9 | Simulation mode: watch a ping cross the ISP | Packets show as ESP between 203.0.113.2 and 198.51.100.2; inner IP header is not visible |

### Redundancy

| # | Test | Expected result |
|---|------|-----------------|
| 10 | `show standby brief` on DS1 and DS2 | DS1 Active for VLAN 10, DS2 Active for VLAN 20 |
| 11 | `show spanning-tree vlan 10` and `vlan 20` | DS1 is root for VLAN 10, DS2 for VLAN 20; exactly one AS1 uplink is blocking per VLAN |
| 12 | Continuous ping from a VLAN 10 PC to a branch PC, then `shutdown` the DS1 SVIs or power DS1 off | A few packets lost, then replies resume through DS2 (HSRP takeover) |
| 13 | Continuous ping, then `shutdown` AS1 Fa0/1 | Short interruption, then traffic uses Fa0/2 (RSTP convergence) |
| 14 | Bring DS1 back | VLAN 10 gateway returns to DS1 (preempt) |

### Layer 2 security

| # | Test | Expected result |
|---|------|-----------------|
| 15 | Plug another switch into an AS1 access port | Port goes err-disabled (BPDU Guard); `show errdisable recovery` or `show interfaces status err-disabled` |
| 16 | Connect a rogue switch with a lower bridge priority to a DS trunk port facing AS1 | `show spanning-tree inconsistentports` lists the port as root-inconsistent (Root Guard) |
| 17 | Attach a rogue DHCP server to an access port | Clients never get an address from it; AS1 drops its offers (`show ip dhcp snooping`) |
| 18 | `show ip dhcp snooping binding` on AS1 | Entries for the PCs that got leases via the trusted path |
| 19 | Move a PC to a different access port, or plug in a third device | Port err-disabled (`show port-security interface fa0/3`) |

## Limits and ideas to extend it

- The HQ edge is a single router. A natural next step is a second edge router and ISP link with HSRP/VRRP or a dynamic routing failover, plus a second tunnel per branch.
- Packet Tracer's IOS support for VRRP is device-dependent. This lab uses HSRP. If your PT version supports VRRP on your L3 switches, swap one VLAN to VRRP and compare behaviour.
- IKEv1 with pre-shared keys is used because it is what Packet Tracer supports well. In a real design, prefer IKEv2 and certificates or a vault-managed PSK.
- Possible add-ons: HSRP interface tracking, an ACL or zone-based firewall at the HQ edge, DMVPN replacing the two static tunnels, NAT for Internet access.

## What to write up in your own words

After testing, add a "Lessons learned" section: what failed first (licence, crypto ACL mismatch and mismatched PSK are the classic ones), how you debugged it (`debug crypto isakmp`, `show crypto ipsec sa`), and what you would improve.
