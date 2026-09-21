# Home Lab Setup Runbook
 
## Overview
This runbook documents the setup order for the homelab. Phases are designed to be completed in order, and each phase builds on the previous.
 
---
 
## Phase 1: Hardware Assembly
- [x] Install DDR5 SODIMM RAM into CWWK mini PC
- [x] Install NVMe SSD into CWWK mini PC
- [x] Verify hardware is recognized on boot
- [x] Install initial drives in Dell PowerEdge T340 (about 2 TB)
- [x] Purchase remaining Ethernet cables as needed
- [x] Clean Dell PowerEdge T340 (dusty)
---
 
## Phase 2: OPNsense Setup
- [x] Download OPNsense ISO and flash to USB
- [x] Install OPNsense on CWWK mini PC
- [x] Access OPNsense web UI (https://192.168.1.1)
- [x] Change default root password
- [x] Confirm interface assignments (ETH0=LAN/igc0, ETH1=WAN/igc1)
- [x] Connect Cox modem to WAN port (ETH1)
- [x] Verify WAN connectivity and internet access
---
 
## Phase 3: Basic Connectivity
- [x] Connect Cisco SG300 to OPNsense LAN (ETH0)
- [x] Connect PC, Xbox, and Ubiquiti U6+ to Cisco SG300
- [x] Adopt U6+ in UniFi Network Application (PC) and UniFi app (iPhone)
- [x] Create basic SSID (HomeNet, no VLANs yet)
- [x] Verify wired devices receive IP from OPNsense DHCP (192.168.1.x)
- [x] Verify internet access on all wired and wireless devices
---
 
## Phase 4: VLAN Configuration
Note: OPNsense 26.1 uses Kea for DHCP. Keep Dnsmasq running until Kea is fully restarted, and keep a safety-net subnet in Kea so connectivity is not lost mid-change.
 
- [x] Create VLAN interfaces in OPNsense (10, 20, 30, 40, 60)
- [x] Configure DHCP server per VLAN in OPNsense (Kea)
- [x] Configure static IP reservations (OPNsense, SG300, Dell T340)
- [x] Configure trunk port on Cisco SG300 to OPNsense (all VLANs)
- [x] Configure trunk port on Cisco SG300 to U6+ (VLANs 20, 30, 40, 60)
- [x] Configure access ports on Cisco SG300 per device (Dell T340 on VLAN 20, untagged)
- [x] Map SSIDs to VLANs in UniFi (HomeNet to 20, MediaNet to 30, IoTNet to 40, GuestNet to 60)
- [x] Create the Private_Networks alias and implement firewall rules per VLAN (see network/firewall-rules.md)
- [x] Verify each VLAN has an allow-to-WAN rule and that pass rules sit above block rules
- [x] Verify devices receive correct IPs per VLAN
- [x] Verify VLAN isolation is working correctly
---
 
## Phase 5: Proxmox & Pi-hole (DNS & Ad Blocking)
- [ ] Power on Dell PowerEdge T340 and update firmware and BIOS
- [ ] Configure the RAID virtual disk on the PERC H330 with the installed drives
- [ ] Download the Proxmox VE ISO and flash to USB
- [ ] Install Proxmox VE on the Dell PowerEdge T340
- [ ] Assign static IP 192.168.20.11 (Trusted VLAN 20)
- [ ] Access the Proxmox web UI (https://192.168.20.11:8006)
- [ ] Configure storage for VM disks and ISO images
- [ ] Deploy Pi-hole as a Proxmox guest (LXC container or VM) at 192.168.20.12
- [ ] Configure OPNsense DHCP to push Pi-hole DNS (192.168.20.12) to VLANs 10, 20, 30
- [ ] Configure OPNsense DHCP to push 1.1.1.1 to VLANs 40, 60
- [ ] Verify the Media VLAN DNS allow rule (port 53 to 192.168.20.12)
- [ ] Verify DNS filtering is working across VLANs
- [ ] Decide whether to add a fallback DNS server, since Pi-hole goes down whenever the T340 is rebooted or rebuilt
---
 
## Phase 6: WireGuard (Remote VPN Access)
- [ ] Install WireGuard package in OPNsense
- [ ] Generate server and client keys
- [ ] Configure WireGuard tunnel in OPNsense
- [ ] Configure firewall rules for VPN traffic
- [ ] Test remote access from outside network
---
 
## Phase 7: Lab VMs
- [ ] Create lab VMs on Proxmox as needed
- [ ] Set up snapshots and backups for VMs
- [ ] Document each lab in the services folder
---
 
## Phase 8: Plex & Media Storage (Future)
- [ ] Add drives for media storage to the Dell PowerEdge T340
- [ ] Deploy Plex Media Server as a Proxmox guest (LXC container or VM) at 192.168.20.13
- [ ] Add Plex firewall rules (VLANs 30 and 40 to 192.168.20.13, port 32400)
- [ ] Test Plex access from Media VLAN devices
---
 
## Phase 9: Suricata (Intrusion Detection)
- [ ] Install Suricata package in OPNsense
- [ ] Configure rulesets
- [ ] Monitor alerts and tune rules
- [ ] Document findings
---
 
## Status
| Phase | Status |
|---|---|
| Phase 1: Hardware Assembly | 🔧 In Progress |
| Phase 2: OPNsense Setup | ✅ Complete |
| Phase 3: Basic Connectivity | ✅ Complete |
| Phase 4: VLAN Configuration | ✅ Complete |
| Phase 5: Proxmox & Pi-hole | 📋 Planned |
| Phase 6: WireGuard | 📋 Planned |
| Phase 7: Lab VMs | 📋 Planned |
| Phase 8: Plex & Media Storage | 📋 Future |
| Phase 9: Suricata | 📋 Future |
