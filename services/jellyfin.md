# Jellyfin

Jellyfin is a free, open source media server. It runs as an LXC container on the Proxmox host and plays movies and shows stored on the SAS RAID 10 array.

Jellyfin was chosen over Plex because it needs no account and keeps every feature, including remote streaming and hardware transcoding, free. It fills the media server slot that was originally planned for Plex.

---

## Container Configuration
| Setting | Value |
|---|---|
| Proxmox Host | Dell PowerEdge T340 (`pve`, 192.168.20.11) |
| Container ID | 101 |
| Hostname | jellyfin |
| Template | Debian 13 |
| Type | Unprivileged LXC |
| Features | Nesting enabled |
| CPU | 2 cores |
| Memory | 4096 MB RAM, 512 MB swap |
| Disk | 16 GB on `local-lvm` (RAID 1 array) |
| Network | `vmbr0`, no VLAN tag (lands on Trusted VLAN 20) |
| IP Address | 192.168.20.13/24 (static, set in Proxmox) |
| Gateway | 192.168.20.1 |
| DNS | Pi-hole (192.168.20.12) |
| Media mount | Host `/mnt/pve/media/library` to container `/media` |
| Web UI | http://192.168.20.13:8096 |

The application lives on the RAID 1 array and the media lives on the RAID 10 array. Keeping them separate means the media can move to bigger drives later by copying the files and repointing one mount.

Swap was left at the 512 MB default. Swap is slow overflow space on disk, and with 4 GB of RAM Jellyfin should not need it.

---

## Storage Layout
```
RAID 10 array (/dev/sdb, ext4)
   │  mounted on the Proxmox host
   ▼
/mnt/pve/media
   └── library
          ├── movies   ──> /media/movies inside the container
          └── shows    ──> /media/shows inside the container
```

The folders and the mount were created from the `pve` node shell before the container's first start:

```
mkdir -p /mnt/pve/media/library/movies
mkdir -p /mnt/pve/media/library/shows
pct set 101 -mp0 /mnt/pve/media/library,mp=/media
```

These commands print nothing when they succeed. To confirm:

```
ls /mnt/pve/media/library
pct config 101
```

The second command should include an `mp0:` line. The mount also appears under **container 101 > Resources**.

See [storage.md](storage.md) for how the array was built.

---

## Install
From the container's console, logged in as root:

```
apt update && apt install -y curl
curl -s https://repo.jellyfin.org/install-debuntu.sh | bash
```

The script adds Jellyfin's official repository and installs the server.

### First-time setup
Browsed to `http://192.168.20.13:8096` and followed the wizard:
- Created the admin account
- Added the libraries below
- Left **Allow remote connections** checked. In Jellyfin, "remote" means any device outside the server's own subnet, which includes the other VLANs. It does not open the server to the internet
- Left **Enable automatic port mapping** unchecked. That option uses UPnP to ask the router to open a port to the internet. Remote access will go through WireGuard instead (Phase 6)

---

## Libraries
| Library | Content Type | Folder |
|---|---|---|
| Movies | Movies | `/media/movies` |
| Shows | Shows | `/media/shows` |

Settings:
- Content type has to be right when the library is created, since it can't be changed afterward
- Preferred language English, country United States
- Trickplay and chapter image extraction left off. They process every video file to build preview thumbnails, which is slow and heavy on the CPU

### File naming
Jellyfin matches artwork and details from the folder and file names:

```
movies/Movie Name (2010)/Movie Name (2010).mkv
shows/Show Name/Season 01/Show Name S01E01.mkv
```

---

## Plugins
| Plugin | Purpose | Status |
|---|---|---|
| Jellyfin Enhanced | Extra interface features, shortcuts, and playback tweaks | ✅ Installed |

Jellyfin Enhanced is installed from its own plugin repository:
1. **Dashboard > Plugins > Manage Repositories > Add**
2. Name `Jellyfin Enhanced`, URL `https://raw.githubusercontent.com/n00bcodr/jellyfin-plugins/main/manifest.json`
3. Installed **Jellyfin Enhanced** from the plugin catalog
4. Restarted Jellyfin and refreshed the browser with Ctrl+F5

The plugin has an optional integration with Seerr, a request manager. It was skipped, since it would mean running another VM for what amounts to a wishlist on a single-user server.

---

## Adding Media
The library is built from discs I own, ripped to MKV files with MakeMKV.

| Format | Drive Needed | Typical Rip Size |
|---|---|---|
| DVD | Any DVD drive (an old Mac's built-in drive works) | 4 to 8 GB |
| Blu-ray | A Blu-ray drive (external USB drive on order) | 25 to 35 GB |

At those sizes the 837 GB array holds roughly 25 Blu-rays or well over 100 DVDs. HandBrake can shrink Blu-ray rips to about a third of their size if space gets tight.

4K UHD discs are skipped for now. They need specific drive firmware to rip, the files run 60 to 80 GB, and HDR video is hard for the server to convert on the fly.

Other sources that work: public domain films from the Internet Archive, DRM-free purchases, and home videos. Purchases from streaming stores can't be used because they are locked to their own apps.

### Copying files to the server
1. Connect to `192.168.20.11` by SFTP as root (WinSCP or FileZilla)
2. Copy into `/mnt/pve/media/library/movies` or `shows`, using the naming above
3. In Jellyfin: **Dashboard > Libraries > Scan All Libraries**

---

## Firewall
Jellyfin sits on VLAN 20, so Trusted devices reach it with no firewall rule. The TVs and streaming devices are on the Media VLAN (30), which is isolated, so they need an allow rule to `192.168.20.13` on TCP port 8096. That rule has not been added yet (see [network/firewall-rules.md](../network/firewall-rules.md)).

---

## Backups
- Container 101 is covered by the nightly 21:00 backup job, which writes to the `media` storage (see [proxmox.md](proxmox.md)). That protects the Jellyfin install, settings, and library metadata
- The bind-mounted media folder is not part of that backup. The media itself is only protected by the RAID 10 array's redundancy, and the original discs are the fallback

---

## Troubleshooting
| Problem | Cause | Fix |
|---|---|---|
| Plugin didn't appear in the catalog | The GitHub download link was pasted as the repository URL | Use the plugin's `manifest.json` URL instead |

---

## Next Steps
- [ ] Rip a first disc and confirm it plays, to test the whole path
- [ ] Add the Media VLAN firewall rule (VLAN 30 to 192.168.20.13, port 8096) and test from a TV
- [ ] Install the Jellyfin app on the Media VLAN devices
- [ ] Set up WireGuard for watching away from home (Phase 6)
- [ ] Move the library to larger drives when they are added (see [storage.md](storage.md))
