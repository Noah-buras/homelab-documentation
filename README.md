# homelab-documentation
Home lab build documenting network setup, VLAN configuration, and self-hosted services for Network+ and CCNA skill development.
 
---
 
## Overview
A home lab built around an Intel N100 mini PC running OPNsense,
a Cisco SG300 managed switch, a Ubiquiti U6+ Wi-Fi 6 access point,
and a Dell PowerEdge T340 server. Designed to develop hands-on
networking skills and serve as a portfolio for Network+ and CCNA
certifications.
 
---
 
## Hardware
 
| Device | Model | Status |
|---|---|---|
| Firewall/Router | CWWK CW-ADLN-4L Mini PC (Intel N100, 4x 2.5GbE) | ✅ Online |
| Switch | Cisco SG300 (L2 mode) | ✅ Online |
| Access Point | Ubiquiti U6+ (Wi-Fi 6) | ✅ Online |
| Server/NAS | Dell PowerEdge T340 (8-bay) | 🔧 In Progress |
| NAS (future) | UGREEN NASync DXP4800 Plus | 📋 Planned |
 
---
 
## Network Topology
 
```
Internet
   │
Cox Modem (ARRIS SB8200)
   │
OPNsense WAN (ETH1 / igc1)
OPNsense LAN (ETH0 / igc0)
   │
[Long Ethernet cable — OPNsense in separate room]
   │
Cisco SG300 (L2 Switch)
   ├── PC 1 (Trusted VLAN 20)
   ├── PC 2 (Trusted VLAN 20)
   ├── Xbox (Media VLAN 30)
   └── Ubiquiti U6+ Access Point
          ├── HomeNet  → VLAN 20 (Trusted)
          ├── MediaNet → VLAN 30 (Media)
          ├── IoTNet   → VLAN 40 (IoT)
          └── GuestNet → VLAN 60 (Guest)
```
 
---
 
## Network Design
 
- **5 VLANs** — Management, Trusted, Media, IoT, Guest
- **Subnet scheme** — `192.168.x.0/24` per VLAN
- **Firewall** — OPNsense with inter-VLAN rules
- **DNS** — Pi-hole in Docker on Dell PowerEdge T340 (192.168.20.11)
### VLAN Table
 
| VLAN ID | Name | Subnet | Gateway | DNS |
|---|---|---|---|---|
| 10 | Management | 192.168.10.0/24 | 192.168.10.1 | Pi-hole (192.168.20.11) |
| 20 | Trusted | 192.168.20.0/24 | 192.168.20.1 | Pi-hole (192.168.20.11) |
| 30 | Media | 192.168.30.0/24 | 192.168.30.1 | Pi-hole (192.168.20.11) |
| 40 | IoT | 192.168.40.0/24 | 192.168.40.1 | 1.1.1.1 (Direct) |
| 60 | Guest | 192.168.60.0/24 | 192.168.60.1 | 1.1.1.1 (Direct) |
 
> VLAN 50 is intentionally skipped.
 
---
 
## Services
 
| Service | Host | Purpose | Status |
|---|---|---|---|
| OPNsense | CWWK Mini PC | Firewall / Router | ✅ Running |
| Pi-hole | Dell PowerEdge T340 (Docker) | DNS / Ad blocking | 📋 Planned |
| WireGuard | OPNsense | Remote VPN access | 📋 Planned |
| Plex | Dell PowerEdge T340 | Media server | 📋 Planned |
| Suricata | OPNsense | Intrusion detection | 📋 Planned |
 
---
 
## Build Progress
 
| Phase | Description | Status |
|---|---|---|
| Phase 1 | Hardware Assembly | 🔧 In Progress |
| Phase 2 | OPNsense Setup | ✅ Complete |
| Phase 3 | Basic Connectivity | ✅ Complete |
| Phase 4 | VLAN Configuration | 📋 Next Up |
| Phase 5 | Pi-hole (DNS & Ad Blocking) | 📋 Planned |
| Phase 6 | WireGuard (Remote VPN) | 📋 Planned |
| Phase 7 | Plex & NAS Services | 📋 Planned |
| Phase 8 | UGREEN NAS | 📋 Future |
| Phase 9 | Suricata (IDS) | 📋 Future |
 
---
 
## Diagrams
See the `/diagrams` folder for logical and physical network diagrams.
 
---
 
## Goals
- Achieve CompTIA Network+ certification
- Build toward CCNA-level skills
- Document everything as a portfolio piece
