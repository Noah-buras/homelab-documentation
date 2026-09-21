# Firewall Rules
 
## Overview
All inter-VLAN traffic is blocked by default. Only explicitly allowed traffic is permitted. Rules are configured in OPNsense and applied per VLAN interface.
 
OPNsense denies anything without a matching pass rule, including internet-bound traffic, so every VLAN interface needs an explicit allow-to-WAN rule. Rules are evaluated top to bottom, so specific allow rules sit above the block rule, and the allow-to-WAN rule sits below it.
 
A firewall alias named `Private_Networks` (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16) is used as the destination for block rules. This blocks other VLANs while leaving internet access to the final allow rule.
 
---
 
## Rules Per Interface
Rules are listed in the order they are evaluated.
 
### VLAN 10 (Management)
| # | Destination | Port | Action | Purpose |
|---|---|---|---|---|
| 1 | Any | Any | Allow | Full access for management |
 
### VLAN 20 (Trusted)
| # | Destination | Port | Action | Purpose |
|---|---|---|---|---|
| 1 | WAN (any non-private) | Any | Allow | Internet access |
 
Trusted and the Dell PowerEdge T340 share VLAN 20, so no firewall rule is needed for Trusted devices to reach Pi-hole or Plex. Access from Trusted to other VLANs is not defined yet and stays blocked.
 
### VLAN 30 (Media)
| # | Destination | Port | Action | Purpose |
|---|---|---|---|---|
| 1 | 192.168.20.11 | 53 (TCP/UDP) | Allow | DNS to Pi-hole |
| 2 | 192.168.20.11 | 32400 (TCP) | Allow | Plex on the T340 |
| 3 | Private_Networks | Any | Block | Fully isolate Media devices |
| 4 | WAN (any non-private) | Any | Allow | Internet access |
 
### VLAN 40 (IoT)
| # | Destination | Port | Action | Purpose |
|---|---|---|---|---|
| 1 | 192.168.20.11 | 32400 (TCP) | Allow | Plex on the T340 |
| 2 | Private_Networks | Any | Block | Fully isolate IoT devices |
| 3 | WAN (any non-private) | Any | Allow | Internet access (DNS goes direct to 1.1.1.1) |
 
### VLAN 60 (Guest)
| # | Destination | Port | Action | Purpose |
|---|---|---|---|---|
| 1 | Private_Networks | Any | Block | Fully isolate Guest network |
| 2 | WAN (any non-private) | Any | Allow | Guest internet access only |
 
---
 
## DNS Rules
| VLAN | DNS Server | IP | Notes |
|---|---|---|---|
| Management (10) | Pi-hole | 192.168.20.11 | Hosted on the Dell PowerEdge T340 |
| Trusted (20) | Pi-hole | 192.168.20.11 | Hosted on the Dell PowerEdge T340 |
| Media (30) | Pi-hole | 192.168.20.11 | Needs the DNS allow rule above, since Media is otherwise isolated |
| IoT (40) | Cloudflare | 1.1.1.1 | Direct, Pi-hole excluded |
| Guest (60) | Cloudflare | 1.1.1.1 | Direct, Pi-hole excluded |
 
---
 
## Notes
- Management VLAN (10) is the only VLAN with access to the OPNsense admin UI and the Cisco SG300 management interface
- Pi-hole and Plex run on the Dell PowerEdge T340 at 192.168.20.11, not on OPNsense itself
- If Pi-hole or Plex move to the UGREEN NAS (192.168.20.10) in Phase 8, update the destination IP in the rules above
- OPNsense DHCP will push the correct DNS server per VLAN automatically
- All rules will be implemented and verified during Phase 4 setup
- Rules will be updated as the lab evolves
 
