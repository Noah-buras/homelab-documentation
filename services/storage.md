# Storage

How the Dell PowerEdge T340's eight drive bays are laid out, how the SAS RAID 10 array was built, and the plan for adding more space later.

---

## Layout
| Array | Drives | RAID Level | Usable | Proxmox Storage | Purpose |
|---|---|---|---|---|---|
| Boot / VM array | 2x 500GB SATA | RAID 1 | ~465GB | `local`, `local-lvm` | Proxmox OS, guest disks, backups |
| Media array | 6x 300GB SAS, 10K rpm, 2.5" | RAID 10 | ~837GB | `media` (directory) | Jellyfin media library |

Both arrays are hardware RAID on the Dell PERC H330. All eight bays are in use.

Inside Proxmox the arrays show up as two disks:
- `/dev/sda`: the RAID 1 array
- `/dev/sdb`: the RAID 10 array, shown as 836.6G with model `PERC H330 Adp`

---

## Installing the SAS Drives
The six SAS drives came in caddies that don't fit the T340's hot-swap bays. Two had been sitting in the bays in the wrong caddies, and the other four waited on compatible trays.

1. Mounted four drives in the new trays
2. Shut the server down from the Proxmox web UI (**pve > Shutdown**) and unplugged it
3. Pulled the two SAS drives that were already installed and moved them into new trays
4. Installed all six drives, plugged the server back in, and powered on

The two SATA drives holding Proxmox were not touched.

### Why the server was powered off
Hot-swap bays are meant for replacing one failed drive in a redundant array. Pulling several members of an array while it is running takes the virtual disk offline and risks corrupting it, so the swap was done with the power off.

---

## Building the RAID 10
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

RAID 10 mirrors the drives in pairs and stripes across the pairs, so six 279 GB drives give about 837 GB usable. It can lose one drive from each pair and keep running.

---

## Checking Drive Health from Proxmox
The drives sit behind the RAID controller, so `smartctl` has to be told to go through it. Run from the `pve` node shell:

```
lsblk -o NAME,SIZE,TYPE,MODEL
for i in {0..7}; do smartctl -H -d megaraid,$i /dev/sda; done
```

The first command confirms the new array exists. The second prints a health result for each physical drive, in order.

Results on 2026-10-02:

| Disk | Type | Result |
|---|---|---|
| 0, 1 | SATA | PASSED |
| 2 | SAS | OK |
| 3 | SAS | **Firmware impending failure, seek error rate too high** |
| 4, 5, 6, 7 | SAS | OK |

To get the details of a single drive:

```
smartctl -i -d megaraid,3 /dev/sda
```

---

## Drive Flagged for Replacement
| Item | Value |
|---|---|
| Controller device ID | 3 |
| Model | Hitachi HUC103030CSS600 (Ultrastar C10K300) |
| Specs | 300 GB, 10K rpm, 2.5", 6 Gb/s SAS, 512-byte sectors |
| Serial | PDVS6EHE |
| Symptom | Bay LED cycles green, amber, off |
| SMART status | Impending failure, seek error rate too high |

A high seek error rate means the heads are struggling to position accurately, which is mechanical wear. The array is still fully redundant, and RAID 10 keeps running if this drive dies because its mirror partner holds a full copy.

### Replacement requirements
- 2.5" SAS, 300 GB or larger (the controller refuses to rebuild onto anything smaller)
- 10K rpm preferred, 6 or 12 Gb/s both work
- 512-byte sectors. Drives pulled from NetApp, EMC, or 3PAR systems are often formatted with 520-byte sectors and won't be accepted
- Not SATA, to avoid mixing drive types in the array

### Replacement procedure
No shutdown needed, since this is a single drive in a redundant array:
1. Confirm the serial number on the drive in the blinking bay
2. Pull it, move the tray onto the new drive, and insert it in the same bay
3. The controller rebuilds onto the new drive
4. Re-run the `smartctl` loop to confirm every drive reports healthy

---

## Adding the Array to Proxmox
The array was first added as an **LVM-Thin** pool, then changed to a **Directory** once it was decided the array would hold media.

| Storage Type | Holds | Fit for this array |
|---|---|---|
| LVM-Thin | Virtual disks for VMs and containers only | Wrong fit. Media files can't be stored on it directly |
| Directory | Ordinary files and folders, plus backups and ISOs | Right fit. A folder can be shared straight into the Jellyfin container |

Steps:
1. Removed the thin pool: **Datacenter > Storage > Remove**, then **pve > Disks > LVM-Thin > More > Destroy** with the cleanup options checked. The confirmation asks for the pool's name. The pool named `data` belongs to the RAID 1 array and must never be destroyed
2. **pve > Disks > Directory > Create: Directory**
3. Disk `/dev/sdb`, filesystem `ext4`, name `media`, **Add Storage** checked

The array is mounted on the host at `/mnt/pve/media`.

---

## Future Upgrade Plan
More media space is planned, with no date set. The constraints:
- All eight bays are full, so adding drives means freeing bays first
- The H330 cannot grow an existing RAID 10, so any change means building a new array

| Option | Change | Result |
|---|---|---|
| Shrink the SAS array | Rebuild it as a 4-drive RAID 10 and add 2x 4 TB in RAID 1 | ~558 GB fast storage plus 4 TB bulk storage |
| Replace the SAS array | Pull all six SAS drives and install 4x 4 TB in RAID 10 | 8 TB usable, two bays free |
| Replace with RAID 5 | Pull all six and install 3 or 4x 4 TB in RAID 5 | 8 to 12 TB usable, with slow writes since the H330 has no cache |

Whichever option is chosen, the media is copied off, the old virtual disk is deleted, the new array is built and added to Proxmox, and the media is copied back. Proxmox and the guests live on the RAID 1 array, so they are not at risk during the change.

Notes for buying drives:
- 4 TB drives are 3.5" and fit the trays without the 2.5" adapter
- SATA or SAS both work on the H330
- CMR drives only. SMR drives perform badly in RAID and can drop out during rebuilds

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

---

## Next Steps
- [ ] Buy a replacement SAS drive and swap out the one flagged for failure
- [ ] Re-check drive health after the rebuild
- [ ] Watch for a good price on a matched pair (or more) of 4 TB CMR drives
- [ ] Decide on the upgrade layout when the drives are in hand
