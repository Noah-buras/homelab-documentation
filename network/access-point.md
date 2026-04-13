#Access point

Wireless network access is provided by Ubiquiti U6+ Access Point, by connecting to a trunk port on Cisco SG300 switch, carrying VLAN's 20 and 60, and is managed via UniFi Network Application running on the NAS

# SSID Maps
|SSID| Maps to VLAN| Subnet| Gateway| Purpose|
|---|---|---|---|---|
 |HomeNet| VLAN 20(Trusted)|192.168.20.0/24| 192.168.20.1| Trusted wireless access| 
|GuestNet| VLAN 60(Guest)| 192.168.60.0/24| 192.168.60.1| Isolated guest access|

| Port | Mode | Allowed VLANs | Connected Device | Purpose |
|---|---|---|---|---|
| Port undetermined | Trunk | 20, 60 | Ubiquiti U6+ | Wireless AP uplink |
