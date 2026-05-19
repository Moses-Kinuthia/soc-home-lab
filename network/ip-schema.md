# Network IP Schema — SOCLAB

## Physical Hosts

| Host | OS | Role |
|---|---|---|
| Machine A | Windows 11 | Runs pfSense + Wazuh VMs |
| Machine B | Windows 10 | Runs DC01 + WKSTNO1 + Kali VMs |

## IP Addressing

| VM | Interface | IP Address | Segment |
|---|---|---|---|
| pfSense | WAN | 192.168.x.x | Home router LAN |
| pfSense | LAN | 10.0.0.1 | Production |
| pfSense | OPT1 (labnet) | 10.0.10.1 | Management |
| pfSense | OPT2 (attack) | 10.0.20.1 | Attack (isolated) |
| Wazuh Manager | OPT1 | 10.0.10.11 | Management |
| DC01 | LAN | 10.0.0.10 | Production |
| WKSTNO1 | LAN | 10.0.0.100 | Production |
| Kali Linux | OPT2 | 10.0.20.11 | Attack (isolated) |

## Segment Rules Summary

| Source → Destination | Policy | Reason |
|---|---|---|
| LAN → OPT1 | Allow (agents only, ports 1514/1515) | Wazuh agent traffic |
| OPT2 → OPT1 | Allow (agents only, ports 1514/1515) | Wazuh agent traffic |
| pfSense → OPT1 | Allow (UDP 514) | Syslog forwarding |
| OPT2 → WAN | Allow + NAT | Kali internet access |
| OPT2 → LAN | Allow (attack sim) + logged | Simulated attack traffic |
| LAN → OPT2 | Block | Default deny |
| OPT2 → OPT1 (general) | Block | Management isolation |
| All other inter-segment | Block | Default deny |

## Domain

| Property | Value |
|---|---|
| Domain name | SOCLAB.local |
| Domain controller | DC01 (10.0.0.10) |
| DNS server | DC01 |
| Simulation accounts | jkamau, gwanjiku, itsupport_backup |
