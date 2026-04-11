# Firewall Rules

## Overview
All inter-VLAN traffic is blocked by default. Only explicitly 
allowed traffic is permitted. Rules are configured in OPNsense.

## Inter-VLAN Rules
| Rule | Source | Destination | Port | Action | Purpose |
|---|---|---|---|---|---|
| Plex Access | 192.168.20.10 | VLAN 30 | 32400 | Allow | NAS Plex to Media VLAN |
| Block IoT | VLAN 40 | Any VLAN | Any | Block | Isolate IoT devices |
| Block Media | VLAN 30 | Any VLAN | Any | Block | Isolate Media devices |
| Allow Internet Access| VLAN 60| WAN| Any| Allow|Guest internet access|
|Block inter-VLAN Access| VLAN 60| Any VLAN| Any| Block| Isolate guest network| 
## DNS Rules
| VLAN | DNS Server | Notes |
|---|---|---|
| Management | 192.168.10.1 | Pi-hole |
| Trusted | 192.168.10.1 | Pi-hole |
| Media | 192.168.10.1 | Pi-hole |
| IoT | 1.1.1.1 | Direct not Pi-hole|
|Guest | 1.1.1.1| Direct, not Pi-hole|

## Notes
- Management VLAN is the only VLAN with access to OPNsense 
  admin UI and Cisco SG300 management interface
- All rules will be implemented and verified during OPNsense setup
- Rules will be updated as the lab evolves
