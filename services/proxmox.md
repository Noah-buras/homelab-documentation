# Proxmox VE

The Dell PowerEdge T340 is the main server in the lab. It runs Proxmox VE, which hosts containers and VMs for services like Pi-hole and future lab machines.

---

## Host Summary
| Item | Value |
|---|---|
| Hardware | Dell PowerEdge T340 (8-bay tower) |
| RAID Controller | Dell PERC H330 (hardware RAID) |
| Hypervisor | Proxmox VE 9.2 (Debian 13 "Trixie" base) |
| Node Name | pve |
| IP Address | 192.168.20.11/24 (static, Trusted VLAN 20) |
| Gateway | 192.168.20.1 (OPNsense) |
| Web UI | https://192.168.20.11:8006 |
| iDRAC | 192.168.1.105 (needs a second Ethernet cable to run alongside the main NIC) |
| Switch Port | Cisco SG300 access port, VLAN 20 untagged |

---

## Storage
| Array | Drives | RAID Level | Usable | Purpose | Status |
|---|---|---|---|---|---|
| Boot / VM array | 2x 500GB SATA | RAID 1 | ~465GB | Proxmox OS, container and VM disks | ✅ Online |
| Data array | 6x 300GB SAS | RAID 10 | ~837GB | VM storage, backups, future media | 📋 Waiting on drive trays |

Proxmox currently uses its default storage on the RAID 1 array:
- `local`: ISO images, container templates, and backups
- `local-lvm`: container and VM disks

### Why hardware RAID instead of ZFS
The PERC H330 would need to be crossflashed to IT mode to pass the drives through for ZFS. That is not officially supported by Dell and adds risk and complexity, so the lab uses the controller's hardware RAID.

### RAID 10 notes
- Only 2 of the 6 SAS drives are seated right now. The drives came in caddies that don't fit the T340's bays, so 4 more compatible trays are needed.
- The H330 cannot expand a RAID 10 after it is created, so the array has to be built with all 6 drives at once.
- The spare PERC 6/i controller I was given is an older generation and is not compatible with the T340's backplane, so it is not used.

---

## Post-Install Setup

### 1. Switch to the free update repositories
A fresh Proxmox install points at the enterprise repositories, which need a paid subscription. Without one, updates fail and the dashboard shows a warning.

In **Node > Updates > Repositories**:
1. Disabled the `pve-enterprise` repository
2. Disabled the `ceph` enterprise repository (Ceph is for multi-node clusters and isn't needed on a single host)
3. Added the **No-Subscription** repository
4. Ran **Refresh** and then **Upgrade** under **Node > Updates**, then rebooted to load the new kernel

The Debian repositories were left enabled since they are free.

### 2. Hardening
- [ ] Create a separate admin account (`noah@pve`, Administrator role on `/`) for day-to-day use, keeping `root@pam` as a break-glass account
- [ ] Enable TOTP two-factor authentication on both accounts and store the recovery keys somewhere safe

### 3. Optional: remove the subscription popup
The "No valid subscription" popup at login is harmless. It can be removed by patching the web UI's JavaScript from the node shell:

```
sed -Ezi.bak "s/(function\(orig_cmd\) \{)/\1\n\torig_cmd\(\);\n\treturn;/g" /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js && systemctl restart pveproxy.service
```

Proxmox updates can undo this, so it may need to be run again after upgrading.

---

## Guests
| ID | Name | Type | IP | Purpose | Docs |
|---|---|---|---|---|---|
| 100 | pihole | LXC (Debian 13) | 192.168.20.12 | DNS and ad blocking | [pihole.md](pihole.md) |

Planned IP pattern for the Trusted VLAN: static server addresses go in `.2` to `.99`, outside the Kea DHCP pool (`.100` to `.200`).

---

## Networking Notes
- The host sits on the Trusted VLAN through a plain access port, so every guest attached to `vmbr0` without a VLAN tag lands on VLAN 20 automatically.
- To put a guest on another VLAN later, `vmbr0` needs to be made VLAN-aware and the SG300 port changed to a trunk (VLAN 20 untagged as the native VLAN, other VLANs tagged). Save the SG300 running-config to startup-config after the change.

---

## Lessons Learned
- **Shut down cleanly before moving the server.** Use the Shutdown button in the web UI, or a short press of the power button (never a long press), and wait for the fans to stop before unplugging.
- **Enterprise repos break updates without a subscription.** Switching to the No-Subscription repo is one of the first things to do after install.
- **A RAID 1 array is not a backup.** It survives a failed drive, not a bad config change or a deleted container. Scheduled backups are still needed.

---

## Next Steps
- [ ] Buy 4 compatible drive trays and build the 6-drive SAS RAID 10
- [ ] Add the RAID 10 array to Proxmox as storage
- [ ] Set up a scheduled backup job (Datacenter > Backup) for all guests
- [ ] Build a general-purpose lab VM and convert it to a template for fast cloning
- [ ] Connect iDRAC with a second Ethernet cable for remote power and console access
- [ ] Put the server on a UPS
