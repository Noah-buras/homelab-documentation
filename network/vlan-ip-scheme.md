# VLAN & IP Scheme
 
## VLAN Table
| VLAN ID | Name | Subnet | Gateway | DNS |
|---|---|---|---|---|
| 10 | Management | 192.168.10.0/24 | 192.168.10.1 | OPNsense Unbound (Pi-hole planned) |
| 20 | Trusted | 192.168.20.0/24 | 192.168.20.1 | Pi-hole (192.168.20.12) |
| 30 | Media | 192.168.30.0/24 | 192.168.30.1 | OPNsense Unbound (Pi-hole planned) |
| 40 | IoT | 192.168.40.0/24 | 192.168.40.1 | 1.1.1.1 (Direct) |
| 60 | Guest | 192.168.60.0/24 | 192.168.60.1 | 1.1.1.1 (Direct) |
 
## Static IP Assignments
| Device | VLAN | IP Address |
|---|---|---|
| OPNsense (Admin UI) | Management (10) | 192.168.10.1 |
| Cisco SG300-28PP | Default LAN (VLAN 1) | 192.168.1.100 (DHCP, moving to 192.168.10.2 on VLAN 10) |
| Ubiquiti U6+ | Management (10) | 192.168.10.3 (Kea reservation) |
| Dell PowerEdge T340 (Proxmox host) | Trusted (20) | 192.168.20.11 |
| Pi-hole (LXC container 100) | Trusted (20) | 192.168.20.12 |
| Jellyfin (LXC container 101) | Trusted (20) | 192.168.20.13 |
| UniFi OS Server (VM 202) | Trusted (20) | 192.168.20.14 |
| lab-01 (VM 201) | Trusted (20) | 192.168.20.21 |
| Dell T340 iDRAC | Default LAN | 192.168.1.105 |
 
## Reserved for Future Use
| Device | VLAN | IP Address | Status |
|---|---|---|---|
| None right now | | | 192.168.20.13 was reserved for a media server and is now used by Jellyfin |
 
## DHCP Devices
| Device | VLAN |
|---|---|
| PC 1 | Trusted (20) |
| PC 2 | Trusted (20) |
| Wireless Devices (personal) | Trusted (20) |
| Smart TV | Media (30) |
| Apple TV | Media (30) |
| Xbox | Media (30) |
| Wireless Devices (media) | Media (30) |
| IoT Devices (future) | IoT (40) |
| Wireless Devices (IoT) | IoT (40), once IoTNet is deployed |
| Wireless Devices (guests) | Guest (60), once GuestNet is deployed |
 
## Notes
- VLAN 50 is intentionally skipped
- The Dell PowerEdge T340 runs Proxmox VE and sits on the Trusted VLAN. The Proxmox web UI is at https://192.168.20.11:8006
- Each Proxmox guest has its own IP. Pi-hole is at 192.168.20.12 and Jellyfin is at 192.168.20.13, not on the host address
- IoT and Guest VLANs use Cloudflare DNS directly, bypassing Pi-hole
- DHCP is served by Kea in OPNsense 26.1
- The VLAN 20 Kea DHCP pool is 192.168.20.100 to 192.168.20.200. Servers use static addresses from .2 to .99 so they never overlap the pool
- The Proxmox host and Pi-hole have static IPs set directly on the device (in the Proxmox installer and in the container's network settings), not DHCP reservations
- Only VLAN 20 uses Pi-hole for now. VLANs 10 and 30 will move to Pi-hole next, now that the wireless setup is finished (see services/pihole.md)
- The U6+ is the first device on the Management VLAN. Its switch port (GE4) uses VLAN 10 as the untagged native VLAN (see network/access-point.md)
- The SG300 is still managed on VLAN 1 at 192.168.1.100. Moving it to 192.168.10.2 is a planned change
- The Proxmox host stays on the Trusted VLAN for now. Moving it to the Management VLAN with a trunk port is a possible later change
