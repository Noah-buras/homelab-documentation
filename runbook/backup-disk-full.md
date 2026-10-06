# Incident: Backup Job Filled the Root Disk

Nightly backups started failing with `No space left on device`. The backup job was writing every guest to the Proxmox root filesystem, and the saved copies used up the whole disk.

---

## Summary
| Item | Value |
|---|---|
| Found | 2026-10-06, from red **Backup Job** entries in the Proxmox task log |
| First failure | 2026-10-01, 22:30 run |
| Affected | Backups for VMs 200, 201, and 202 from Oct 1. Container 101 (Jellyfin) from Oct 5 |
| Root cause | Backups for all guests stored on `local`, a 94 GB root filesystem, with retention set higher than the disk could hold |
| Service impact | None. Every guest kept running. The risk was a week with no fresh backup of the UniFi VM and a host with 0 bytes free |
| Fixed | 2026-10-06. Root disk back to 19% used |

---

## Symptoms
- The last 7 **Backup Job** tasks ended with `Error: job errors`
- The job log showed the same error for each failing guest:

```
zstd: error 70 : Write error : cannot write block : No space left on device
ERROR: Backup of VM 202 failed - vma_queue_write: write error - Broken pipe
```

- The Pi-hole container still backed up fine each night because its archive is under 300 MB, which made the job look partly healthy
- Cleanup after the failed Jellyfin backup also failed, because removing the temporary snapshot needs to write to the same full disk:

```
snapshot 'vzdump' was not (fully) removed - lvremove snapshot 'pve/snap_vm-101-disk-0_vzdump' error: ... fclose failed: No space left on device
```

---

## Diagnosis
Run from the `pve` node shell:

```
df -h /
ls -lah /var/lib/vz/dump/
du -xh / --max-depth=2 2>/dev/null | sort -h | tail -20
findmnt | grep -i media
pct config 101
```

What each one showed:

| Check | Result |
|---|---|
| `df -h /` | `/dev/mapper/pve-root` 94G, 100% used, 0 available |
| `ls` of the dump folder | 85G of backup files |
| `du` | 88G under `/var/lib`, confirming the backups were the only large item |
| `findmnt` | `/mnt/pve/media` mounted from `/dev/sdb1`, so media files had not been written to the root disk by mistake |
| `pct config 101` | `lock: snapshot-delete` and `parent: vzdump` left behind by the failed cleanup |

Where the 85G went:

| Guest | Copies | Size each | Total |
|---|---|---|---|
| 200 ubuntu-base (template) | 7 | 3.8G | ~27G |
| 201 lab-01 (stopped) | 7 | 3.8G | ~27G |
| 202 unifi | 5 | 5.7G | ~29G |
| 100 pihole, 101 jellyfin | 13 | 0.3G to 0.4G | ~4G |

---

## Root Cause
Three settings combined:

1. **Storage was `local`.** On a default install, `local` is a folder on the root filesystem (`/var/lib/vz`), not a separate disk. The guest disks live on `local-lvm`, which is a different pool, so the 465 GB array looked like it had plenty of room.
2. **Selection was all guests.** That included a template and a stopped VM. Neither changes, so the job saved the same 3.8G image every night.
3. **Retention was 7 daily and 4 weekly.** Pruning only starts once a guest has more copies than the policy keeps. The disk filled before any guest reached that point, and after that no new VM backup could finish, so nothing was ever pruned.

---

## Fix

### 1. Free space
The seven copies of each stopped guest were identical, so the September ones were deleted and the Oct 1 copy of each was kept:

```
cd /var/lib/vz/dump
rm vzdump-qemu-200-2026_09_* vzdump-qemu-201-2026_09_*
df -h /
```

### 2. Clear the stuck snapshot on container 101
```
pct unlock 101
pct delsnapshot 101 vzdump
pct config 101
```

The `lock` and `parent` lines were gone afterward. This has to come after step 1, since removing the snapshot is what failed for lack of space in the first place.

### 3. Move backups to the RAID 10 array
1. **Datacenter > Storage > media > Edit**, added **Backup** to the Content list
2. **Datacenter > Backup > Edit** on the job:
   - Storage: `media`
   - Selection mode: Include selected VMs, with 100, 101, and 202 ticked
   - Retention: Keep Daily 3, Keep Weekly 2

On older Proxmox versions the content type is labeled "VZDump backup file". On 9.2 it is just "Backup".

### 4. Test and clean up
1. **Run now** on the job. It finished with OK and the new files appeared under **pve > media > Backups**
2. Deleted the old backups under **pve > local > Backups**, keeping the Oct 1 copies of `ubuntu-base` and `lab-01`
3. Confirmed with `df -h /` (19% used) and `pct config 101` (no lock)

---

## Lessons Learned
- **`local` is the root disk.** Filling it does more than stop backups. A host with no free space on `/` can fail in other ways, so backups should go somewhere that can fill up safely.
- **Do the math before setting retention.** Size per backup, times guests, times copies kept. Here that came to well over the 94G available.
- **Don't back up guests that never change on a schedule.** A template or a powered-off VM needs one backup, not one a night.
- **A full disk blocks its own cleanup.** Pruning and snapshot removal both need a little free space, so the job could not recover without manual help.
- **One green guest can hide a failing job.** Pi-hole kept succeeding while everything after it failed. The job's overall status is what counts.
- **Check the mount before blaming the data.** `findmnt` ruled out the other common cause, which is files written to an unmounted mount point and landing on the root disk.
- **Watch the task log or set up notifications.** The job failed for five days before it was noticed. The failure emails went to `mail-to-root`, which nobody reads.

---

## Next Steps
- [ ] Confirm the next scheduled run finishes with OK
- [ ] Point backup notifications at a real mailbox
- [ ] Copy the `dump` folder to an external drive or another machine now and then, since the backups share an array with the media
