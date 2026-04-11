# VLAN & IP Scheme

## VLAN Table
| VLAN ID | Name | Subnet | Gateway | Pi-hole DNS |
|---|---|---|---|---|
| 10 | Management | 192.168.10.0/24 | 192.168.10.1 | Yes |
| 20 | Trusted | 192.168.20.0/24 | 192.168.20.1 | Yes |
| 30 | Media | 192.168.30.0/24 | 192.168.30.1 | Yes |
| 40 | IoT | 192.168.40.0/24 | 192.168.40.1 | No (1.1.1.1) |
|60 | Guest| 192.168.60.0/24| 192.168.60.1| No (1.1.1.1)|

## Static IP Assignments
| Device | VLAN | IP Address |
|---|---|---|
| OPNsense (Admin UI) | Management | 192.168.10.1 |
| Cisco SG300 (Management) | Management | 192.168.10.2 |
| UGREEN NAS | Trusted | 192.168.20.10 |

## DHCP Devices
| Device | VLAN |
|---|---|
| PC 1 | Trusted |
| PC 2 | Trusted |
|Wirless Devices| Trusted|
| Smart TV | Media |
| Apple TV | Media |
| Xbox | Media |
| IoT Devices (future) | IoT |
|Wirless Devices| Guest|
