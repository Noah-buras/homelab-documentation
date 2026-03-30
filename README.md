# homelab-documentation
Home lab build documenting network setup, VLAN configuration, and self-hosted services for Network+ and CCNA skill development.
---

## Overview
A home lab built around an Intel N100 mini PC running OPNsense, 
a Cisco SG300-28P managed switch, and a UGREEN NAS. Designed to 
develop hands-on networking skills and serve as a portfolio for 
Network+ and CCNA certifications.

---

## Hardware
| Device | Model |
|---|---|
| Firewall/Router | Intel N100 Mini PC (4x 2.5GbE) |
| Switch | Cisco SG300-28P |
| NAS | UGREEN NASync DXP4800 Plus |

---

## Network Design
- **4 VLANs** — Management, Trusted, Media, IoT
- **Subnet scheme** — 192.168.x.0/24 per VLAN
- **Firewall** — OPNsense with inter-VLAN rules
- **DNS** — Pi-hole running on OPNsense

## Services
| Service | Purpose | Status |
|---|---|---|
| OPNsense | Firewall / Router | Planned |
| Pi-hole | DNS / Ad blocking | Planned |
| WireGuard | Remote VPN access | Planned |
| Plex | Media server | Planned |
| Suricata | Intrusion detection | Planned |

---

## Diagrams
See the `/diagrams` folder for logical and physical network diagrams.

---

## Goals
- Achieve CompTIA Network+ certification
- Build toward CCNA-level skills
- Document everything as a portfolio piece
