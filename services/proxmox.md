# Proxmox VE

The Dell PowerEdge T340 is the main server in the lab. It runs Proxmox VE, which hosts containers and VMs for services like Pi-hole, Jellyfin, and the lab machines.

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
| Media array | 6x 300GB SAS (10K) | RAID 10 | ~837GB | Jellyfin media library | ✅ Online (one drive flagged for replacement) |

Proxmox storage:
- `local` (RAID 1): ISO images and container templates
- `local-lvm` (RAID 1): container and VM disks
- `media` (RAID 10): directory storage mounted at `/mnt/pve/media`, holding the Jellyfin library and guest backups

The full build steps, drive health checks, and upgrade plan are in [storage.md](storage.md).

### Why hardware RAID instead of ZFS
The PERC H330 would need to be crossflashed to IT mode to pass the drives through for ZFS. That is not officially supported by Dell and adds risk and complexity, so the lab uses the controller's hardware RAID.

### RAID 10 notes
- The SAS drives came in caddies that don't fit the T340's bays, so the array waited on compatible trays. With all 6 drives in the new trays, the array was built in one step.
- The H330 cannot expand a RAID 10 after it is created, so the array had to be built with all 6 drives at once. It can still rebuild onto a replacement drive in the same bay.
- One of the six drives reports a predicted failure and is due to be replaced (see [storage.md](storage.md)).
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
- [x] Create a separate admin account (`noah@pve`, Administrator role on `/`) for day-to-day use, keeping `root@pam` as a break-glass account
- [x] Enable TOTP two-factor authentication on both accounts and store the recovery keys somewhere safe

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
| 101 | jellyfin | LXC (Debian 13) | 192.168.20.13 | Media server | [jellyfin.md](jellyfin.md) |
| 200 | ubuntu-base | VM template (Ubuntu Server 26.04) | DHCP | Base image for lab VMs | [ubuntu-template.md](ubuntu-template.md) |
| 201 | lab-01 | VM (clone of 200) | 192.168.20.21 | General-purpose lab machine | [ubuntu-template.md](ubuntu-template.md) |
| 202 | unifi | VM (full clone of 200) | 192.168.20.14 | UniFi OS Server, wireless controller | [unifi.md](unifi.md) |

IP pattern for the Trusted VLAN: static addresses go in `.2` to `.99`, outside the Kea DHCP pool (`.100` to `.200`). Core services use the low `.10`s, and lab VMs start at `.21`.

ID pattern: LXC containers use 100 and up, VMs and templates use 200 and up.

---

## Backups
| Setting | Value |
|---|---|
| Location | Datacenter > Backup |
| Schedule | Daily at 21:00 |
| Selection | Guests 100 (pihole), 101 (jellyfin), and 202 (unifi) |
| Storage | `media` (SAS RAID 10 array, files in `/mnt/pve/media/dump`) |
| Mode | Snapshot (guests keep running during backup) |
| Compression | ZSTD |
| Retention | Keep 3 daily, 2 weekly |
| Notes | `{{guestname}}` |

The job originally backed up every guest to `local`, which sits on the 94 GB Proxmox root filesystem. That filled the disk within a week and the job started failing. The full story and the fix are in [backup-disk-full.md](../runbook/backup-disk-full.md).

What changed on 2026-10-06:
- Backups go to the `media` storage, so they no longer share a disk with the Proxmox OS
- The template (200) and the stopped lab VM (201) are left out of the nightly job. They don't change, so one manual backup of each is kept on `local`
- Retention was cut down to fit the three guests that are still covered

New guests are no longer picked up automatically. Anything worth protecting has to be ticked in the job's selection after it is built.

The backups now live on a different array than the guests, which is better than before, but they share that array with the media library and nothing is copied off the server yet. They protect against a broken guest or a bad update, not against losing the server.

---

## Networking Notes
- The host sits on the Trusted VLAN through a plain access port, so every guest attached to `vmbr0` without a VLAN tag lands on VLAN 20 automatically.
- To put a guest on another VLAN later, `vmbr0` needs to be made VLAN-aware and the SG300 port changed to a trunk (VLAN 20 untagged as the native VLAN, other VLANs tagged). Save the SG300 running-config to startup-config after the change.

---

## Lessons Learned
- **Shut down cleanly before moving the server.** Use the Shutdown button in the web UI, or a short press of the power button (never a long press), and wait for the fans to stop before unplugging.
- **Enterprise repos break updates without a subscription.** Switching to the No-Subscription repo is one of the first things to do after install.
- **Use the node's Shutdown button, not Bulk Shutdown.** Bulk Shutdown only stops the guests and leaves the server powered on.
- **Power down before pulling drives that are in an array.** Hot-swap is for replacing one failed drive in a redundant array, not for removing several members at once.
- **Pi-hole goes down with the server.** Every device on VLAN 20 loses DNS while the T340 is off, so plan maintenance around that.
- **Check where backups land and how much room is there.** The default `local` storage is the root filesystem. Multiply backup size by guests by retention before trusting a schedule, and look at the task log now and then, because a failing nightly job is easy to miss.
- **A RAID 1 array is not a backup.** It survives a failed drive, not a bad config change or a deleted container. Scheduled backups are still needed.

---

## Next Steps
- [x] Buy compatible drive trays and build the 6-drive SAS RAID 10
- [x] Add the RAID 10 array to Proxmox as storage (`media` directory)
- [ ] Replace the SAS drive that reports a predicted failure
- [x] Set up a scheduled backup job (Datacenter > Backup) for all guests
- [x] Move backups off the RAID 1 array (now on the `media` storage)
- [ ] Copy backups somewhere off the server
- [x] Build a general-purpose lab VM and convert it to a template for fast cloning (see [ubuntu-template.md](ubuntu-template.md))
- [ ] Connect iDRAC with a second Ethernet cable for remote power and console access
- [ ] Put the server on a UPS
