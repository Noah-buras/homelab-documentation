# Access Point
 
Wireless network access is provided by a Ubiquiti U6+ (Wi-Fi 6) Access Point. It connects to a trunk port on the Cisco SG300 switch carrying VLANs 20, 30, 40, and 60. The AP is currently managed through the UniFi Network Application on a PC and the UniFi Network app on an iPhone. A self-hosted UniFi Network Server is planned once the server hardware is running services.
 
---
 
## SSID Map
| SSID | VLAN | Subnet | Gateway | Purpose |
|---|---|---|---|---|
| HomeNet | VLAN 20 (Trusted) | 192.168.20.0/24 | 192.168.20.1 | Personal trusted wireless access |
| MediaNet | VLAN 30 (Media) | 192.168.30.0/24 | 192.168.30.1 | Wireless media devices |
| IoTNet | VLAN 40 (IoT) | 192.168.40.0/24 | 192.168.40.1 | Isolated IoT device access |
| GuestNet | VLAN 60 (Guest) | 192.168.60.0/24 | 192.168.60.1 | Isolated guest wireless access |
 
---
 
## Switch Port Configuration
| Port | Mode | Allowed VLANs | Connected Device | Purpose |
|---|---|---|---|---|
| Port TBD | Trunk | 20, 30, 40, 60 | Ubiquiti U6+ | Wireless AP uplink |
 
---
 
## Notes
- VLAN 10 (Management) is intentionally excluded from the trunk, so there is no wireless management access
- IoTNet and GuestNet are isolated from all other VLANs and have internet access only
- Only HomeNet is configured today (no VLAN tagging yet). The other SSIDs and VLAN mappings are set up in Phase 4
- Migration to a self-hosted UniFi Network Server is planned, with the host and timing still to be decided
- Port assignment on the Cisco SG300 will be updated once the trunk is configured
