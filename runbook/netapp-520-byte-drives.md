# Incident: New 4 TB Drives Rejected by the RAID Controller

Two refurbished 4 TB SAS drives would not join a RAID array on the Dell PERC H330. The controller menus froze and then refused to create the virtual disk. The drives were healthy but still formatted with 520-byte sectors from a NetApp storage array, and were fixed by reformatting them to 512-byte sectors from Linux.

---

## Summary
| Item | Value |
|---|---|
| Date | 2026-10-12 to 2026-10-13 |
| Drives | 2x HGST Ultrastar 7K6000 4TB SAS (HUS726040AL5210), bought as eBay Refurbished |
| Listed as | 512e |
| Actually | NetApp firmware (`X477_HAKPE04TA07`), 520-byte sectors |
| Root cause | The drives were pulled from a NetApp array and never reformatted. The H330 only accepts 512-byte (or 4K) sectors |
| Fix | Set the drives to Non-RAID, reformatted each to 512-byte sectors with `sg_format`, then built the RAID 1 |
| Time to fix | About 7 hours, almost all of it the format running unattended |
| Data lost | None. The drives were new and empty, and the other arrays were never touched |

---

## Symptoms
1. Creating a RAID 1 with **Full initialization** accepted the confirmation, then froze the setup screen for several minutes
2. After it recovered, opening **Virtual Disk Management** froze it again for over 20 minutes
3. After a reset, the RAID 1 had not been saved. Trying again with **Fast** initialization returned:

```
Invalid error status.
```

4. Opening **Physical Disk Management** froze the menu as well, until both drives were re-seated and the server rebooted

---

## Diagnosis

### The controller's view
Once Physical Disk Management loaded, the end of each line gave it away:

```
Physical Disk 00:01:00: HDD, SATA, 465.250GB, Online, (512B)
Physical Disk 00:01:01: HDD, SATA, 465.250GB, Online, (512B)
Physical Disk 00:01:02: HDD, SAS, 3.695TB, Ready, (0KB)
Physical Disk 00:01:03: HDD, SAS, 3.695TB, Ready, (0KB)
Physical Disk 00:01:04: HDD, SAS, 278.875GB, Ready, (512B)
...
```

- **`(0KB)`**: every working drive reports its sector size. The new drives reported none, meaning the controller couldn't use their format
- **3.695 TB**: a 4 TB drive with 512-byte sectors shows as about 3.638 TB. The larger figure matches the same number of blocks at 520 bytes each

### The operating system's view
With both drives set to Non-RAID so Linux could see them directly:

```
lsblk -o NAME,SIZE,MODEL
NAME   SIZE   MODEL
sda      0B   X477_HAKPE04TA07
sdb      0B   X477_HAKPE04TA07
sdc  465.3G   PERC H330 Adp
sdd  557.8G   PERC H330 Adp
```

- **0B**: Linux won't use a disk with 520-byte sectors either
- **X477_HAKPE04TA07**: a NetApp part number, so these were NetApp-firmware drives under an HGST label

`sg_format` confirmed it before making any change:

```
NETAPP    X477_HAKPE04TA07  NA04   peripheral_type: disk [0x0]
      << supports protection information>>
  Number of blocks=7814037168 [0x1d1c0beb0]
  Block size=520 [0x208]
```

---

## Root Cause
Enterprise storage arrays from NetApp, EMC, and others format their drives with 520-byte sectors. The extra 8 bytes per sector hold the array's own checksums. When those drives are pulled and resold, they keep that format unless the seller reformats them.

The PERC H330 only works with 512-byte or 4K sectors. It couldn't read the drives properly, waited on long timeouts, and that is what made the setup menus hang.

The listing photo said 512e, which is the format the drive model ships with from HGST. It did not reflect how these specific drives were formatted.

---

## Fix

### 1. Hand the drives to Linux
In **F2 System Setup > Device Settings > PERC H330 > Physical Disk Management**, set both 4 TB drives to **Convert to Non-RAID** so they pass straight through to the OS. The other arrays were left alone, and the 4-drive SAS RAID 10 was built at the same time.

### 2. Identify the drives
From the `pve` node shell:

```
apt install -y sg3-utils tmux
lsblk -o NAME,SIZE,MODEL
```

The pass-through drives showed up first (`sda` and `sdb`) and pushed the boot array to `sdc`. Checking the model column before formatting was essential, since the format wipes whatever device it is pointed at.

### 3. Reformat to 512-byte sectors
Each format was run in its own `tmux` window so a closed browser tab couldn't interrupt it:

```
tmux new -s fmt
sg_format --format --size=512 --fmtpinfo=0 /dev/sdb
```

The second drive was added to the same session from another shell:

```
tmux new-window -t fmt 'sg_format --format --size=512 --fmtpinfo=0 /dev/sda'
```

- `--size=512` sets the new sector size
- `--fmtpinfo=0` turns off protection information, which the drives supported and which the H330 doesn't need

`sg_format` waits 15 seconds before starting, which is the last chance to cancel with Ctrl+C. After "Format unit has started", the drive does the work itself.

Progress can be checked from any shell:

```
sg_requests --progress /dev/sda
sg_requests --progress /dev/sdb
```

Both drives ran in parallel and took about 7 hours. The format slowed down near the end, since the inner tracks of a disk hold less data per rotation.

### 4. Confirm the new format
```
sg_readcap -l /dev/sda
sg_readcap -l /dev/sdb
```

Both reported `Logical block length=512`.

### 5. Build the array
1. Rebooted into the PERC menu. Both drives now showed **(512B)**
2. Converted both back to **RAID Capable**
3. Created a **RAID 1** named `media` with **Fast** initialization
4. Booted Proxmox and checked each drive's health:

```
smartctl -a -d megaraid,2 /dev/sda | grep -iE "health|power on|grown defect"
smartctl -t long -d megaraid,2 /dev/sda
```

| Drive | Power-on hours | Grown defects | SMART health |
|---|---|---|---|
| 2 | 10,993 | 0 | OK |
| 3 | 23,310 | 0 | OK |

Once the format was fixed, the drives turned out to have low hours and no remapped sectors.

---

## Lessons Learned
- **Check the model string, not the label.** `lsblk` or `smartctl -i` shows the firmware the drive is really running. A NetApp part number means 520-byte sectors are likely.
- **"512e" in a listing describes the drive model, not necessarily the drive you get.** Ask the seller to confirm the current sector format for anything pulled from a storage array.
- **A frozen RAID menu usually means a drive problem, not a menu problem.** The controller waits on a drive it can't talk to, and the whole interface stalls.
- **Look at the sector size column.** `(0KB)` next to a drive in Physical Disk Management means the controller can't use it.
- **The H330 can pass drives through.** Non-RAID mode gave Linux direct access, which made the fix possible without a separate HBA.
- **Confirm the device letter before a destructive command.** Pass-through drives took over `sda` and `sdb`, and the boot array moved to `sdc`.
- **Run long jobs in `tmux`.** Firefox grabs Ctrl+B, so new windows were started with `tmux new-window` from a second shell instead of the keyboard shortcut.
- **A low-level format is limited by the platters, not the interface.** Rewriting every sector of a 4 TB 7200 RPM drive takes hours whether it is SAS or SATA.
- **Fixing it beat returning it.** The drives were otherwise exactly what was needed, and the fix was a useful hands-on exercise in SCSI tools.
