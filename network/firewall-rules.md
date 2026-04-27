# Firewall Rules
 
## Overview
All inter-VLAN traffic is blocked by default. Only explicitly allowed traffic is permitted. Rules are configured in OPNsense and applied per VLAN interface.
 
---
 
## Inter-VLAN Rules
| Rule | Source | Destination | Port | Action | Purpose |
|---|---|---|---|---|---|
| Allow Plex | VLAN 20, 30, 40 | 192.168.20.10 | 32400 | Allow | Access Plex on NAS from Media, Trusted, IoT |
| Block IoT | VLAN 40 | Any VLAN | Any | Block | Fully isolate IoT devices |
| Block Media | VLAN 30 | Any VLAN | Any | Block | Fully isolate Media devices |
| Allow Guest Internet | VLAN 60 | WAN | Any | Allow | Guest internet access only |
| Block Guest inter-VLAN | VLAN 60 | Any VLAN | Any | Block | Fully isolate Guest network |
| Allow Management | VLAN 10 | Any VLAN | Any | Allow | Full access for management |
 
---
 
## DNS Rules
| VLAN | DNS Server | IP | Notes |
|---|---|---|---|
| Management (10) | Pi-hole | 192.168.20.10 | Hosted on NAS |
| Trusted (20) | Pi-hole | 192.168.20.10 | Hosted on NAS |
| Media (30) | Pi-hole | 192.168.20.10 | Hosted on NAS |
| IoT (40) | Cloudflare | 1.1.1.1 | Direct, Pi-hole excluded |
| Guest (60) | Cloudflare | 1.1.1.1 | Direct, Pi-hole excluded |
 
---
 
## Notes
- Management VLAN (10) is the only VLAN with access to OPNsense admin UI and Cisco SG300 management interface
- Pi-hole runs as a Docker container on the NAS at 192.168.20.10 — not on OPNsense itself
- OPNsense DHCP will push the correct DNS server per VLAN automatically
- All rules will be implemented and verified during Phase 2 setup
- Rules will be updated as the lab evolves
