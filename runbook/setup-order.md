# Home Lab Setup Runbook

## Overview
This runbook documents the setup order for my homelab.


---

## Phase 1 — Hardware Assembly
- [ ] Watch model-specific YouTube tutorial for mini PC
- [ ] Install Crucial DDR5 SODIMM RAM into mini PC
- [ ] Install WD Blue SN570 NVMe SSD into mini PC
- [ ] Verify hardware is recognized on boot
- [ ] Purchase Ethernet cables neccesary
- [ ] Purchase UGREEN NASync DXP4800 Plus NAS

---

## Phase 2 — OPNsense Setup
- [ ] Download OPNsense ISO and flash to USB
- [ ] Install OPNsense on mini PC
- [ ] Configure WAN interface (Cox modem in bridge mode)
- [ ] Verify internet connectivity
- [ ] Configure LAN interface
- [ ] Access OPNsense web UI

---

## Phase 3 — VLAN Configuration
- [ ] Connect mini PC to Cisco SG300 via console cable
- [ ] Configure VLANs on Cisco SG300 (10, 20, 30, 40)
- [ ] Configure trunk port between SG300 and OPNsense
- [ ] Create VLAN interfaces in OPNsense
- [ ] Configure DHCP server per VLAN
- [ ] Configure static IPs (OPNsense, SG300, NAS)
- [ ] Verify devices receive correct IPs per VLAN
- [ ] Implement and verify firewall rules

---

## Phase 4 — Pi-hole
- [ ] Install Pi-hole on OPNsense mini PC
- [ ] Configure VLANs 10, 20, 30 to use Pi-hole DNS
- [ ] Configure VLAN 40 to use 1.1.1.1 directly
- [ ] Verify DNS filtering is working across VLANs

---

## Phase 5 — WireGuard
- [ ] Install WireGuard package in OPNsense
- [ ] Generate server and client keys
- [ ] Configure WireGuard tunnel in OPNsense
- [ ] Configure firewall rules for VPN traffic
- [ ] Test remote access from outside network

---

## Phase 6 — Plex & NAS
- [ ] Set up UGREEN NAS (Once Purchased)
- [ ] Install Plex Media Server on NAS
- [ ] Verify Plex firewall rule (192.168.20.10 → VLAN 30, port 32400)
- [ ] Test Plex access from Media VLAN devices

---

## Phase 7 — Suricata (Future)
- [ ] Install Suricata package in OPNsense
- [ ] Configure rulesets
- [ ] Monitor alerts and tune rules
- [ ] Document findings

---

## Status
| Phase | Status |
|---|---|
| Phase 1 — Hardware Assembly | In Progress |
| Phase 2 — OPNsense Setup | Planned |
| Phase 3 — VLAN Configuration | Planned |
| Phase 4 — Pi-hole | Planned |
| Phase 5 — WireGuard | Planned |
| Phase 6 — Plex & NAS | Planned |
| Phase 7 — Suricata | Planned |
