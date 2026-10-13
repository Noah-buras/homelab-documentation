# Storage

How the Dell PowerEdge T340's eight drive bays are laid out, how each array was built, and how drive health is checked.

---

## Layout
| Array | Drives | RAID Level | Usable | Proxmox Storage | Purpose |
|---|---|---|---|---|---|
| Boot / VM array | 2x 500GB SATA | RAID 1 | ~465GB | `local`, `local-lvm` | Proxmox OS, guest disks |
| Media array | 2x 4TB SAS, 7.2K rpm, 3.5" | RAID 1 | ~3.6TB | `media` (directory) | Jellyfin media library, future photo storage |
| Backup array | 4x 300GB SAS, 10K rpm, 2.5" | RAID 10 | ~558GB | `backup` (directory) | Nightly guest backups |

All three arrays are hardware RAID on the Dell PERC H330. All eight bays are in use.

Inside Proxmox the arrays show up as three disks, each with model `PERC H330 Adp`:

| Disk | Size | Array |
|---|---|---|
| `/dev/sda` | 465.3G | Boot RAID 1 |
| `/dev/sdb` | 557.8G | `backup` RAID 10 |
| `/dev/sdc` | 3.6T | `media` RAID 1 |

> Device letters are not fixed. Any drive set to Non-RAID (pass-through) is detected first and pushes the arrays down the alphabet. Always run `lsblk -o NAME,SIZE,MODEL` before using a device name in a command.

### Why this layout
- **Media and backups are on separate arrays.** Before the rebuild, backups shared the SAS array with the media library, so one failed array would have taken both. Now a problem with the media array leaves the backups intact, and the reverse.
- **The failing SAS drive is gone.** Shrinking the SAS array to four drives freed two bays for the 4 TB pair and let the drive flagged for failure retire instead of being replaced.
- **The media array has room to grow into.** At about 3.6 TB it holds the Jellyfin library and is large enough for a photo library (Immich, planned).

This still is not an off-server backup. A fire, theft, power surge, or controller failure would take every array at once. See Next Steps.

---

## Original Build: 6-Drive SAS RAID 10 (2026-10-02)
This was the first media array. It was replaced by the current layout on 2026-10-12, but the steps are kept since the same process applies to any future array.

### Installing the SAS drives
The six SAS drives came in caddies that don't fit the T340's hot-swap bays. Two had been sitting in the bays in the wrong caddies, and the other four waited on compatible trays.

1. Mounted four drives in the new trays
2. Shut the server down from the Proxmox web UI (**pve > Shutdown**) and unplugged it
3. Pulled the two SAS drives that were already installed and moved them into new trays
4. Installed all six drives, plugged the server back in, and powered on

The two SATA drives holding Proxmox were not touched.

Hot-swap bays are meant for replacing one failed drive in a redundant array. Pulling several members of an array while it is running takes the virtual disk offline and risks corrupting it, so the swap was done with the power off.

### Building the array
Done in **F2 System Setup > Device Settings > PERC H330** at the server's own display.

After the swap, Physical Disk Management showed:

| Drives | State | Meaning |
|---|---|---|
| 2x SATA | Online | The existing RAID 1 array, unaffected |
| 4x SAS | Ready | Blank and available |
| 1x SAS | Foreign | Carrying leftover RAID information from an earlier array |
| 1x SAS | Non-RAID | Set to pass straight through to the OS instead of joining an array |

Cleanup and build:
1. **Configuration Management > Manage Foreign Configuration > Clear Foreign Configuration** to wipe the leftover RAID information from the foreign drive
2. **Configuration Management > Convert to RAID Capable** for the Non-RAID drive
3. Confirmed all six SAS drives showed **Ready**
4. **Configuration Management > Create Virtual Disk**: RAID 10, all six SAS drives, default settings, Fast initialization

> Clear **Foreign** Configuration only affects the foreign drive. The general Clear Configuration option would have wiped the RAID 1 array's configuration as well.

RAID 10 mirrors the drives in pairs and stripes across the pairs, so six 279 GB drives gave about 837 GB usable.

### Adding it to Proxmox
The array was first added as an **LVM-Thin** pool, then changed to a **Directory** once it was decided the array would hold media.

| Storage Type | Holds | Fit for this array |
|---|---|---|
| LVM-Thin | Virtual disks for VMs and containers only | Wrong fit. Media files can't be stored on it directly |
| Directory | Ordinary files and folders, plus backups and ISOs | Right fit. A folder can be shared straight into the Jellyfin container |

On 2026-10-06 the **Backup** content type was enabled on `media` so the nightly backup job could move off the root disk. See [backup-disk-full.md](../runbook/backup-disk-full.md).

---

## The Failing Drive
The first health check on 2026-10-02 flagged one of the six SAS drives:

| Item | Value |
|---|---|
| Controller device ID (at the time) | 3 |
| Model | Hitachi HUC103030CSS600 (Ultrastar C10K300) |
| Specs | 300 GB, 10K rpm, 2.5", 6 Gb/s SAS, 512-byte sectors |
| Serial | PDVS6EHE |
| Symptom | Bay LED cycles green, amber, off |
| SMART status | Impending failure, seek error rate too high |

A high seek error rate means the heads are struggling to position accurately, which is mechanical wear. Instead of buying a replacement, the drive was pulled during the 2026-10-12 rebuild along with one healthy SAS drive. It is labeled as failed and kept out of the server.

---

## Rebuild: 4 TB Mirror and Backup Array (2026-10-12)

### Choosing the drives
Several used 4 TB SAS listings were compared before buying. What mattered for the PERC H330:

| Listing | Problem |
|---|---|
| Seagate Exos 7E8, 4Kn, SED | Self-encrypting drive. The H330 can't manage the encryption, so a drive still locked by its old controller would be unusable without a factory wipe |
| NetApp X375A | NetApp drives usually ship with 520-byte sectors, which the H330 rejects |
| WD RE WD4001FYYG | Compatible, but an older generation with likely high hours |
| HGST Ultrastar 7K6000, sold as-is | Right drive, no returns and no health data |
| **HGST Ultrastar 7K6000 HUS726040AL5210, eBay Refurbished** | **Chosen.** Listed as 512e, free returns, $112 for the pair |

### Steps
1. Checked what was on the old array: the library folder was empty, so only the backups (17 GB) mattered
2. Ran a fresh backup of guests 100, 101, and 202 to `local` as a temporary copy, then disabled the backup job
3. Stopped container 101 and set it not to start on boot, since its media mount pointed at the old array
4. Removed the `media` storage (**Datacenter > Storage > media > Remove**) and stopped its mount with `systemctl disable --now mnt-pve-media.mount`
5. Shut down, pulled the failing drive and one healthy SAS drive, and installed the two 4 TB drives in their trays
6. In the PERC menu, deleted only the 6-drive RAID 10 virtual disk and built a **4-drive RAID 10** from the remaining SAS drives, named `backup`
7. Creating the 4 TB RAID 1 failed. The new drives turned out to be formatted with 520-byte sectors and had to be reformatted first. See [netapp-520-byte-drives.md](../runbook/netapp-520-byte-drives.md)
8. After the reformat, converted both 4 TB drives back to **RAID Capable** and built a **RAID 1** named `media`, Fast initialization
9. Booted Proxmox and removed the old mount unit so the `media` name could be reused:
   ```
   rm /etc/systemd/system/mnt-pve-media.mount
   systemctl daemon-reload
   ```
10. **pve > Disks > Directory > Create: Directory** twice, both ext4 with **Add Storage** checked:
    - The 3.6T disk as `media`
    - The 557.8G disk as `backup`, then set its Content to **Backup** only under **Datacenter > Storage**
11. Recreated `/mnt/pve/media/library/movies` and `shows`, set container 101 to start on boot, and started it
12. Pointed the backup job at `backup`, re-enabled it, ran it, and deleted the temporary copies on `local`

The controller names match the Proxmox storage names (`media` and `backup`), so the F2 menus, `lsblk`, and these docs all use the same words.

Because the new `media` storage reused the old name and path, Jellyfin's mount (`/mnt/pve/media/library`) kept working with no change to the container.

### New drive health
| Controller ID | Model | Power-on hours | Grown defects | SMART health |
|---|---|---|---|---|
| 2 | NetApp X477_HAKPE04TA07 (HGST HUS726040AL5210) | 10,993 | 0 | OK |
| 3 | NetApp X477_HAKPE04TA07 (HGST HUS726040AL5210) | 23,310 | 0 | OK |

Long self-tests were started on both drives after the rebuild. SAS long tests run inside the drive in the background, so the array stayed in use while they ran.

---

## Checking Drive Health from Proxmox
The drives sit behind the RAID controller, so `smartctl` has to be told to go through it. Run from the `pve` node shell:

```
lsblk -o NAME,SIZE,TYPE,MODEL
for i in {0..7}; do smartctl -H -d megaraid,$i /dev/sda; done
```

With `-d megaraid,N`, any disk on the controller works as the doorway and the number picks the physical drive. If `lsblk` shows the device letters have shifted, use any current `PERC H330 Adp` disk.

To identify which physical drive is which:

```
for i in {0..7}; do echo "== $i"; smartctl -i -d megaraid,$i /dev/sda | grep -E "Product|capacity"; done
```

Current controller IDs:

| ID | Drive |
|---|---|
| 0, 1 | 500GB SATA (boot RAID 1) |
| 2, 3 | 4TB SAS (`media` RAID 1) |
| 4 to 7 | 300GB SAS (`backup` RAID 10) |

Hours, defects, and self-test results for one drive:

```
smartctl -a -d megaraid,2 /dev/sda | grep -iE "health|power on|grown defect"
smartctl -l selftest -d megaraid,2 /dev/sda
```

To run a long self-test:

```
smartctl -t long -d megaraid,2 /dev/sda
```

A 4 TB drive takes 8 hours or more.

---

## Buying Drives for the H330
- **512-byte sectors (512n or 512e).** 4Kn works for a data array, but every drive in that array has to be 4Kn
- **Check the model string, not just the label.** A drive can carry an HGST label and still run NetApp firmware with 520-byte sectors. The listing photo for these drives said 512e
- **No self-encrypting drives (SED).** The H330 can't manage drive encryption
- **CMR only.** SMR drives perform badly in RAID and can drop out during rebuilds
- **SATA or SAS both work.** SAS drives don't work in USB docks or enclosures, so they can't double as an external backup drive
- **Buy from a listing with returns.** Test the drives as soon as they arrive, while returns are still open

---

## Lessons Learned
- **Power off before pulling more than one drive from an array.** Hot-swap covers a single failed drive, not a planned reshuffle.
- **Keep drives in their original slots when re-seating them.** The controller can usually find an array anyway, but matching slots avoids a foreign configuration prompt.
- **Foreign and Non-RAID states are leftovers, not faults.** Used drives often arrive with old RAID information or in pass-through mode, and both are cleared from the controller menu.
- **A blinking green, amber, off pattern is worth checking.** A quick cycle means predicted failure. A slow cycle (about 3 seconds green, 3 amber, 6 off) means a rebuild was stopped.
- **SMART data is still readable behind a hardware RAID controller** with `smartctl -d megaraid,N`.
- **Long commands can break when pasted into the Proxmox shell.** If a paste wraps onto several lines, the shell runs the pieces separately. Shorter one-line commands avoid it.
- **Pick the storage type by what will live on it.** LVM-Thin for virtual disks, Directory for files.
- **The standalone iDRAC port needs its own cable.** Without one, controller setup has to be done at a monitor. The T340 only outputs VGA, so an HDMI monitor needs a VGA-to-HDMI adapter that converts in that direction.
- **Keep backups on a different array than the data.** Shrinking the SAS array and giving it to backups fixed that without buying more drives.
- **Reusing a storage name keeps mounts working.** Naming the new array `media` again meant Jellyfin needed no changes.
- **A `(0KB)` sector size in Physical Disk Management means the controller can't use the drive.** It was the first clear sign of the 520-byte format.
- **A Full initialization makes the PERC menus hang.** The controller answers status requests slowly while it zeroes a large drive. Fast initialization plus a SMART long test checks the drive without tying up the server.

---

## Next Steps
- [ ] Confirm both 4 TB long self-tests finished with "Completed" and no errors
- [ ] Copy backups and photos somewhere off the server (external USB drive or a cloud bucket like Backblaze B2)
- [ ] Deploy Immich for photos on the `media` array
- [ ] Connect iDRAC with a second Ethernet cable for remote management
