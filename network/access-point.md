# Access Point
 
Wireless network access is provided by a Ubiquiti U6+ (Wi-Fi 6) Access Point. It connects to a trunk port on the Cisco SG300 switch carrying VLANs 20, 30, 40, and 60. The AP is managed via the UniFi Network app (iPhone) and will later be managed via UniFi Network Server hosted on the NAS.
 
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
- VLAN 10 (Management) is intentionally excluded from the trunk — no wireless management access
- IoTNet and GuestNet are fully isolated — no inter-VLAN access, internet only
- UniFi controller is currently the iPhone app — will migrate to NAS-hosted UniFi Network Server in Phase 3
- Port assignment on the Cisco SG300 will be updated once physically connected
 
