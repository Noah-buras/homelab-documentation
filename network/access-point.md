# Access Point

Wireless access is provided by a Ubiquiti U6+ (Wi-Fi 6) access point. The AP is managed by a self-hosted UniFi OS Server running as a VM on the Proxmox host (see [services/unifi.md](../services/unifi.md)). The AP lives on the Management VLAN, and each SSID is tagged onto its own VLAN.

---

## AP Summary
| Item | Value |
|---|---|
| Model | Ubiquiti U6+ (Wi-Fi 6) |
| Management IP | 192.168.10.3 (Kea DHCP reservation, Management VLAN 10) |
| Controller | UniFi OS Server at 192.168.20.14 (VM 202 on the T340) |
| Inform URL | `http://192.168.20.14:8080/inform` (set with Inform Host Override) |
| Switch Port | Cisco SG300-28PP GE4 (trunk) |
| Power | PoE from the SG300-28PP |

---

## SSID Map
| SSID | VLAN | Subnet | Status | Notes |
|---|---|---|---|---|
| HomeNet | 20 (Trusted) | 192.168.20.0/24 | ✅ Live | Personal devices. DNS from Pi-hole |
| MediaNet | 30 (Media) | 192.168.30.0/24 | ✅ Live | TVs, streaming boxes, Xbox. Isolated from other VLANs |
| IoTNet | 40 (IoT) | 192.168.40.0/24 | 📋 VLAN ready, SSID not deployed | Planned settings: 2.4 GHz only, WPA2 |
| GuestNet | 60 (Guest) | 192.168.60.0/24 | 📋 VLAN ready, SSID not deployed | Planned settings: Client Device Isolation on, WPA2 |

IoTNet and GuestNet aren't broadcast because nothing uses them yet, and every extra SSID adds beacon traffic that uses airtime. The VLANs, DHCP scopes, firewall rules, and switch tagging are already in place, so either SSID can be turned on in under a minute.

No SSID is attached to UniFi's **Default** network. Default means untagged on the AP's port, and untagged on GE4 is the Management VLAN, so an SSID there would put wireless clients on the management network.

---

## Switch Port Configuration
| Port | Mode | VLAN Membership | Connected Device |
|---|---|---|---|
| GE1 | Trunk | 1UP, 10T, 20T, 30T, 40T, 60T | OPNsense LAN (igc0) |
| GE4 | Trunk | 10UP, 20T, 30T, 40T, 60T | Ubiquiti U6+ |

U = untagged, T = tagged, P = PVID. On GE4, VLAN 10 is the native (untagged) VLAN, so the AP's own traffic lands on Management, while the SSID traffic arrives tagged.

---

## How Management Works Across VLANs
The AP (VLAN 10) and the controller (VLAN 20) are on different subnets. This works because of two things:

1. **Inform Host Override.** The controller tells the AP to check in at `192.168.20.14` instead of relying on local discovery. The AP sends a check-in to `http://192.168.20.14:8080/inform` every few seconds, and config changes come back in the reply.
2. **Management firewall rule.** VLAN 10 has a pass-all rule, so the AP's check-ins are routed through OPNsense to VLAN 20. OPNsense tracks the connection, so the replies are allowed back automatically. The controller never has to start a connection to the AP.

Layer 2 discovery only matters for first-time adoption, which is why the AP was adopted on VLAN 20 before moving to VLAN 10 (see the steps below).

---

## Migration to the Self-Hosted Controller
The AP was originally managed by the UniFi Network Application on a PC, which was lost when that PC was wiped. It was reset and adopted fresh on the new server.

1. Added a Kea reservation for the AP's MAC: `192.168.10.3` on the Management subnet
2. Temporarily changed GE4 so VLAN 20 was untagged (`20UP, 30T, 40T, 60T`), putting the AP on the same subnet as the controller
3. Factory reset the AP with the pinhole button
4. Adopted it on the new controller, with Inform Host Override already set
5. Changed GE4 to its final config (VLAN 10 untagged, VLAN 20 tagged) and saved running-config to startup-config
6. Power cycled the AP. It came back on 192.168.10.3 and showed Online in the controller
7. Created the HomeNet (VLAN 20) and MediaNet (VLAN 30) SSIDs. HomeNet kept its old password so devices reconnected on their own

---

## Verification
| Test | Expected Result |
|---|---|
| Phone on HomeNet | IP in 192.168.20.x, DNS server 192.168.20.12 |
| Phone on MediaNet | IP in 192.168.30.x, internet works |
| From MediaNet, open `https://192.168.20.14:11443` | Fails (Media is isolated) |
| Controller Devices page | U6+ Online at 192.168.10.3 |

---

## Troubleshooting
| Problem | Cause | Fix |
|---|---|---|
| AP not adoptable after a reset, then LED cycling white, blue, off | The AP had dropped into TFTP recovery mode | A plain power cycle (no button) brought it back, and it showed as adoptable |
| SG300 **Join VLAN** popup shows empty lists | The old SG300 web UI doesn't render that dialog in modern browsers | Use **VLAN Management > Port to VLAN** instead |
| Changing GE4 to Access mode failed: "port belongs to multiple VLANs" | Access mode only allows one VLAN, and GE4 was already a trunk | Keep it a trunk and just change which VLAN is untagged |
| No **Excluded** option for VLAN 1 on Port to VLAN | VLAN 1 is the default VLAN and is handled differently | Setting another VLAN as untagged + PVID removes the port from VLAN 1 automatically |
| Lost access to the switch web UI after editing GE1 | The change affected VLAN 1, which carries the switch's management IP | Rebooting the switch restored its saved startup-config |
| Lease looked like 192.168.10.103 | The narrow IP column in the Kea leases page wrapped `192.168.10.` and `3` onto two lines | Widen the column or check the MAC before assuming the reservation failed |
| AP offline after moving to VLAN 10 | Nothing had ever used VLAN 10 before, so its path had never been tested | Confirm the Management interface has a pass rule in OPNsense and that VLAN 10 is tagged on GE1 |

---

## Next Steps
- [ ] Move the Xbox to MediaNet after checking IPv6 on the Media VLAN (the Xbox relies heavily on IPv6, and the VLAN interfaces don't have IPv6 configured yet)
- [ ] Deploy IoTNet and GuestNet if they're ever needed
- [ ] Move the SG300's own management IP from VLAN 1 (192.168.1.100) to VLAN 10 (192.168.10.2)
