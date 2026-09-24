# homelab-documentation
Home lab build documenting network setup, VLAN configuration, and self-hosted services for Network+ and CCNA skill development.
 
---
 
## Overview
A home lab built around an Intel N100 mini PC running OPNsense, a Cisco SG300 managed switch, a Ubiquiti U6+ Wi-Fi 6 access point, and a Dell PowerEdge T340 server running Proxmox. Designed to develop hands-on networking and systems skills and serve as a portfolio for Network+ and CCNA certifications.
 
---
 
## Hardware
 
| Device | Model | Status |
|---|---|---|
| Firewall/Router | CWWK CW-ADLN-4L Mini PC (Intel N100, 4x 2.5GbE) | ✅ Online |
| Switch | Cisco SG300 (L2 mode) | ✅ Online |
| Access Point | Ubiquiti U6+ (Wi-Fi 6) | ✅ Online |
| Main Server | Dell PowerEdge T340 (8-bay, Proxmox host) | ✅ Online (storage expansion in progress) |
 
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
   ├── 4 port switch
          ├── PC 1 (Trusted VLAN 20)
          ├── PC 2 (Trusted VLAN 20)
          └── Dell PowerEdge T340 (Proxmox, Trusted VLAN 20)
                 └── Pi-hole LXC (192.168.20.12)
   └── Ubiquiti U6+ Access Point
          ├── HomeNet  → VLAN 20 (Trusted)
          ├── MediaNet → VLAN 30 (Media)
                 └── Xbox (Media VLAN 30)
          ├── IoTNet   → VLAN 40 (IoT)
          └── GuestNet → VLAN 60 (Guest)
```
 
---
 
## Network Design
 
- **5 VLANs** — Management, Trusted, Media, IoT, Guest
- **Subnet scheme** — `192.168.x.0/24` per VLAN
- **Firewall** — OPNsense with inter-VLAN rules
- **DNS**: Pi-hole running as an LXC container on the T340 (192.168.20.12). Currently serving the Trusted VLAN (20) only
### VLAN Table
 
| VLAN ID | Name | Subnet | Gateway | DNS |
|---|---|---|---|---|
| 10 | Management | 192.168.10.0/24 | 192.168.10.1 | OPNsense Unbound (Pi-hole planned) |
| 20 | Trusted | 192.168.20.0/24 | 192.168.20.1 | Pi-hole (192.168.20.12) |
| 30 | Media | 192.168.30.0/24 | 192.168.30.1 | OPNsense Unbound (Pi-hole planned) |
| 40 | IoT | 192.168.40.0/24 | 192.168.40.1 | 1.1.1.1 (Direct) |
| 60 | Guest | 192.168.60.0/24 | 192.168.60.1 | 1.1.1.1 (Direct) |
 
> VLAN 50 is intentionally skipped.
 
---
 
## Services
 
| Service | Host | Purpose | Status |
|---|---|---|---|
| OPNsense | CWWK Mini PC | Firewall / Router | ✅ Running |
| Proxmox VE | Dell PowerEdge T340 | Virtualization for lab VMs and services ([docs](services/proxmox.md)) | ✅ Running |
| Pi-hole | LXC container on the T340 | DNS / Ad blocking ([docs](services/pihole.md)) | ✅ Running (VLAN 20) |
| WireGuard | OPNsense | Remote VPN access | 📋 Planned |
| Lab VMs | Proxmox guests on the T340 | Hands-on labbing ([template docs](services/ubuntu-template.md)) | ✅ Running |
| Plex | Proxmox guest on the T340 | Media server (after storage is added) | 📋 Future |
| Suricata | OPNsense | Intrusion detection | 📋 Future |
---
 
## Build Progress
 
| Phase | Description | Status |
|---|---|---|
| Phase 1 | Hardware Assembly | 🔧 In Progress |
| Phase 2 | OPNsense Setup | ✅ Complete |
| Phase 3 | Basic Connectivity | ✅ Complete |
| Phase 4 | VLAN Configuration | ✅ Complete  |
| Phase 5 | Proxmox & Pi-hole (DNS & Ad Blocking) | 🔧 In Progress |
| Phase 6 | WireGuard (Remote VPN) | 📋 Planned |
| Phase 7 | Lab VMs | 🔧 In Progress |
| Phase 8 | Plex & Media Storage | 📋 Future |
| Phase 9 | Suricata (IDS) | 📋 Future |
 

---
 
## Diagrams
See the `/diagrams` folder for logical and physical network diagrams.
 
---
 
## Service Docs
- [Proxmox VE](services/proxmox.md): T340 host setup, storage, and post-install steps
- [Pi-hole](services/pihole.md): container config, install, and troubleshooting
- [Ubuntu Server Template](services/ubuntu-template.md): VM template, SSH key hardening, and cloning steps
 
---
 
## Goals
- Achieve CompTIA Network+ certification
- Build toward CCNA-level skills
- Document everything as a portfolio piece
