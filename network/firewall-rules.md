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
 
The Proxmox host (192.168.20.11) and its guests share VLAN 20 with Trusted devices, so no firewall rule is needed for Trusted devices to reach the Proxmox web UI, Pi-hole, or Plex. Access from Trusted to other VLANs is not defined yet and stays blocked.
 
### VLAN 30 (Media)
| # | Destination | Port | Action | Purpose |
|---|---|---|---|---|
| 1 | 192.168.20.12 | 53 (TCP/UDP) | Allow | DNS to Pi-hole (add when VLAN 30 moves to Pi-hole) |
| 2 | 192.168.20.13 | 32400 (TCP) | Allow | Plex (add when Plex is deployed, Phase 8) |
| 3 | Private_Networks | Any | Block | Fully isolate Media devices |
| 4 | WAN (any non-private) | Any | Allow | Internet access |
 
### VLAN 40 (IoT)
| # | Destination | Port | Action | Purpose |
|---|---|---|---|---|
| 1 | 192.168.20.13 | 32400 (TCP) | Allow | Plex (add when Plex is deployed, Phase 8) |
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
| Management (10) | Pi-hole (planned) | 192.168.20.12 | Uses OPNsense Unbound for now |
| Trusted (20) | Pi-hole | 192.168.20.12 | Live. LXC container on the T340, same subnet so no rule needed |
| Media (30) | Pi-hole (planned) | 192.168.20.12 | Uses OPNsense Unbound for now. Needs the DNS allow rule above when switched, since Media is otherwise isolated |
| IoT (40) | Cloudflare | 1.1.1.1 | Direct, Pi-hole excluded |
| Guest (60) | Cloudflare | 1.1.1.1 | Direct, Pi-hole excluded |
 
---
 
## Notes
- Management VLAN (10) is the only VLAN with access to the OPNsense admin UI and the Cisco SG300 management interface
- Proxmox (192.168.20.11), Pi-hole (192.168.20.12), and later Plex (192.168.20.13) live on the Dell PowerEdge T340, not on OPNsense
- The Plex rules are added only when Plex is deployed
- OPNsense DHCP will push the correct DNS server per VLAN automatically
- All rules will be implemented and verified during Phase 4 setup
- Rules will be updated as the lab evolves
