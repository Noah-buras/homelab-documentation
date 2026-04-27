# VLAN & IP Scheme
 
## VLAN Table
| VLAN ID | Name | Subnet | Gateway | DNS |
|---|---|---|---|---|
| 10 | Management | 192.168.10.0/24 | 192.168.10.1 | Pi-hole (192.168.20.10) |
| 20 | Trusted | 192.168.20.0/24 | 192.168.20.1 | Pi-hole (192.168.20.10) |
| 30 | Media | 192.168.30.0/24 | 192.168.30.1 | Pi-hole (192.168.20.10) |
| 40 | IoT | 192.168.40.0/24 | 192.168.40.1 | 1.1.1.1 (Direct) |
| 60 | Guest | 192.168.60.0/24 | 192.168.60.1 | 1.1.1.1 (Direct) |
 
## Static IP Assignments
| Device | VLAN | IP Address |
|---|---|---|
| OPNsense (Admin UI) | Management (10) | 192.168.10.1 |
| Cisco SG300 | Management (10) | 192.168.10.2 |
| UGREEN NASync DXP4800 Plus | Trusted (20) | 192.168.20.10 |
| Dell PowerEdge T340 | Trusted (20) | 192.168.20.11 |
 
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
| Wireless Devices (IoT) | IoT (40) |
| Wireless Devices (guests) | Guest (60) |
 
## Notes
- VLAN 50 is intentionally skipped
- Pi-hole runs as a Docker container on the NAS (192.168.20.10)
- IoT and Guest VLANs use Cloudflare DNS directly, bypassing Pi-hole
- Static IPs are assigned via DHCP reservation in OPNsense
