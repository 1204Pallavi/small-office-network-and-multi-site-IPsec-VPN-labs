# Network Projects

Hands-on Cisco networking labs built in Cisco Packet Tracer, with full device configs, IP plans, and verification checklists.

| # | Project | Key topics |
|---|---------|-----------|
| 1 | [Small Office Network](01-small-office-network/) | VLANs, inter-VLAN routing, OSPF, DHCP, DNS, NAT/PAT, ACLs, port security, SSH hardening, wireless guest AP |
| 2 | [Secure Multi-Site Enterprise Network](02-multisite-ipsec-vpn/) | Site-to-site IPsec + GRE tunnels, OSPF over tunnels, HSRP, RSTP, Root Guard, BPDU Guard, DHCP snooping, sticky port security |

## Repo layout

```
network-projects/
├── 01-small-office-network/
│   ├── README.md          # overview, topology, build steps, test checklist
│   ├── ip-addressing.md   # VLANs, subnets, device interfaces
│   ├── configs/           # one text file per device
│   └── topology/          # put your .pkt file and screenshots here
└── 02-multisite-ipsec-vpn/
    └── (same structure)
```

## Notes

- All IP addresses use private ranges (RFC 1918) or documentation ranges (RFC 5737). Nothing here maps a real network.
- All passwords in configs are the placeholder `CHANGE_ME`. Lab only.
- Everything is for simulation. Only test networks you own or are authorised to test.
