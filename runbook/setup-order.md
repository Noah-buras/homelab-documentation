# Home Lab Setup Runbook
 
## Overview
This runbook documents the setup order for the homelab. Phases are designed to be completed in order — each phase builds on the previous.
 
---
 
## Phase 1 — Hardware Assembly
- [x] Install DDR5 SODIMM RAM into CWWK mini PC
- [x] Install NVMe SSD into CWWK mini PC
- [x] Verify hardware is recognized on boot
- [ ] Purchase remaining Ethernet cables as needed
- [ ] Clean Dell PowerEdge T340 (dusty)
- [ ] Purchase and install drives for Dell PowerEdge T340 (8-bay, currently empty)
---
 
## Phase 2 — OPNsense Setup
- [x] Download OPNsense ISO and flash to USB
- [x] Install OPNsense on CWWK mini PC
- [x] Access OPNsense web UI (https://192.168.1.1)
- [x] Change default root password
- [x] Confirm interface assignments (ETH0=LAN/igc0, ETH1=WAN/igc1)
- [ ] Connect Cox modem to WAN port (ETH1) — pending Cox installation
- [ ] Verify WAN connectivity and internet access
- [ ] Run through OPNsense setup wizard
---
 
## Phase 3 — Basic Connectivity
- [ ] Connect Cisco SG300 to OPNsense LAN (ETH0)
- [ ] Connect PC, Xbox, and Ubiquiti U6+ to Cisco SG300
- [ ] Adopt U6+ in UniFi Network app (iPhone)
- [ ] Create basic SSID (no VLANs yet)
- [ ] Verify all wired and wireless devices receive IP from OPNsense DHCP (192.168.1.x)
- [ ] Verify internet access on all devices
---
 
## Phase 4 — VLAN Configuration
- [ ] Create VLAN interfaces in OPNsense (10, 20, 30, 40, 60)
- [ ] Configure DHCP server per VLAN in OPNsense
- [ ] Configure static IP reservations (OPNsense, SG300, NAS devices)
- [ ] Configure trunk port on Cisco SG300 to OPNsense (all VLANs)
- [ ] Configure trunk port on Cisco SG300 to U6+ (VLANs 20, 30, 40, 60)
- [ ] Configure access ports on Cisco SG300 per device
- [ ] Map SSIDs to VLANs in UniFi (HomeNet→20, MediaNet→30, IoTNet→40, GuestNet→60)
- [ ] Implement and verify firewall rules per VLAN
- [ ] Verify devices receive correct IPs per VLAN
- [ ] Verify VLAN isolation is working correctly
---
 
## Phase 5 — Pi-hole (DNS & Ad Blocking)
- [ ] Power on and configure Dell PowerEdge T340
- [ ] Choose and install NAS OS (TrueNAS / Unraid / Linux TBD)
- [ ] Assign static IP to Dell PowerEdge T340 (192.168.20.11)
- [ ] Install Docker on NAS OS
- [ ] Deploy Pi-hole as a Docker container
- [ ] Configure OPNsense DHCP to push Pi-hole DNS to VLANs 10, 20, 30
- [ ] Configure OPNsense DHCP to push 1.1.1.1 to VLANs 40, 60
- [ ] Verify DNS filtering is working across VLANs
---
 
## Phase 6 — WireGuard (Remote VPN Access)
- [ ] Install WireGuard package in OPNsense
- [ ] Generate server and client keys
- [ ] Configure WireGuard tunnel in OPNsense
- [ ] Configure firewall rules for VPN traffic
- [ ] Test remote access from outside network
---
 
## Phase 7 — Plex & NAS Services
- [ ] Install Plex Media Server on Dell PowerEdge T340 (Docker or native)
- [ ] Verify Plex firewall rule (NAS to VLAN 30, port 32400)
- [ ] Test Plex access from Media VLAN devices
- [ ] Set up additional self-hosted services as needed
---
 
## Phase 8 — UGREEN NAS (Future)
- [ ] Purchase UGREEN NASync DXP4800 Plus
- [ ] Assign static IP 192.168.20.10
- [ ] Migrate or expand NAS services from Dell PowerEdge T340
- [ ] Update Pi-hole DNS IP in OPNsense if migrated
---
 
## Phase 9 — Suricata (Intrusion Detection)
- [ ] Install Suricata package in OPNsense
- [ ] Configure rulesets
- [ ] Monitor alerts and tune rules
- [ ] Document findings
---
 
## Status
| Phase | Status |
|---|---|
| Phase 1 — Hardware Assembly | 🔧 In Progress |
| Phase 2 — OPNsense Setup | 🔧 In Progress |
| Phase 3 — Basic Connectivity | ⏳ Pending (Cox modem installation) |
| Phase 4 — VLAN Configuration | 📋 Planned |
| Phase 5 — Pi-hole | 📋 Planned |
| Phase 6 — WireGuard | 📋 Planned |
| Phase 7 — Plex & NAS Services | 📋 Planned |
| Phase 8 — UGREEN NAS | 📋 Future |
| Phase 9 — Suricata | 📋 Future |
