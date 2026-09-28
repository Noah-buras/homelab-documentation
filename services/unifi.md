# UniFi OS Server

A self-hosted UniFi OS Server running as a VM on Proxmox. It manages the Ubiquiti U6+ access point, replacing the UniFi Network Application that used to run on a PC. UniFi OS Server is Ubiquiti's current self-hosting platform and replaces the older standalone Network Application.

---

## VM Summary
| Setting | Value |
|---|---|
| VM ID | 202 |
| Name | unifi |
| Source | Full clone of the Ubuntu Server template (VM 200, see [ubuntu-template.md](ubuntu-template.md)) |
| OS | Ubuntu Server 26.04 LTS |
| CPU | 2 cores, type `host` |
| Memory | 4096 MB, ballooning off |
| Disk | 32 GB on `local-lvm` |
| Network | `vmbr0`, no VLAN tag (Trusted VLAN 20) |
| IP Address | 192.168.20.14/24 (static, netplan) |
| DNS | Pi-hole (192.168.20.12) |
| Start at boot | Yes, startup order 2 (after Pi-hole) |
| Web UI | https://192.168.20.14:11443 |
| Network app | UniFi Network 10.5.67 |

UniFi OS Server runs in Podman and needs a full VM rather than an LXC container. Ubiquiti's minimum is 2 GB of RAM, so 4 GB leaves room to grow.

---

## Install

### 1. Clone and prepare the VM
1. Cloned the template as a **Full Clone** (VM 202, `unifi`). Proxmox defaults to a linked clone for templates, which stays tied to the template's disk
2. Set memory to 4096 MB with ballooning off, and enabled **Start at boot**
3. Set the hostname and a static IP from the Proxmox console, using the netplan steps in [ubuntu-template.md](ubuntu-template.md)
4. Updated the system and took a Proxmox snapshot (`pre-unifi`) as a rollback point before installing

### 2. Install UniFi OS Server
```
sudo apt install -y podman slirp4netns
podman --version        # needs 4.9.3 or newer
wget -O uos-installer "<Linux x64 link from ui.com/download>"
chmod +x uos-installer
sudo ./uos-installer
```
The installer link comes from **ui.com/download > UniFi OS Server**. The T340 is x86-64, so it needs the **Linux x64** build, not arm64. The file is about 750 MB, so a small download means the link was wrong.

### 3. First-time setup
- Browsed to `https://192.168.20.14:11443` and accepted the self-signed certificate warning
- Named the server `unifi-homelab` and created a **local admin account**. Signing in with a UI.com account failed with an ISP error, so the server runs local-only. Remote access through unifi.ui.com isn't needed for the lab

---

## Key Settings
| Setting | Location | Value |
|---|---|---|
| Inform Host Override | Network > Devices > Device Updates and Settings > Device SSH Settings | `192.168.20.14` |
| Device Auto-Update | Same panel | Daily at 3 AM (after the 2 AM Proxmox backup) |
| Device SSH credentials | Same panel | Set by the controller on adoption (kept out of the docs) |

**Inform Host Override** is what lets the AP live on a different VLAN than the controller. Without it, the AP would try to reach the controller by local discovery or the `unifi` hostname, and neither works across VLANs. See [network/access-point.md](../network/access-point.md) for how management traffic flows.

---

## Networks and SSIDs
OPNsense handles routing and DHCP, so the UniFi networks only need a name and VLAN ID.

| UniFi Network | VLAN | SSID | Status |
|---|---|---|---|
| Default | untagged (VLAN 10 on the switch) | none | Not used for SSIDs |
| Trusted | 20 | HomeNet | ✅ Live |
| Media | 30 | MediaNet | ✅ Live |
| IoT | 40 | IoTNet | 📋 Not deployed |
| Guest | 60 | GuestNet | 📋 Not deployed |

---

## Backups
- **Proxmox:** VM 202 is covered by the nightly 02:00 backup job for all guests (see [proxmox.md](proxmox.md)). This can restore the entire server.
- **UniFi backup file:** scheduled cloud backups need a UI.com account, so a backup is downloaded by hand from **Settings > Control Plane > Backups > Download** after major changes and kept on the PC.
- VM 202 is never cloned after UniFi is installed. A clone would reuse the server's identity and authentication tokens.

---

## Troubleshooting
| Problem | Cause | Fix |
|---|---|---|
| Shell showed `>` prompts and nothing ran | A backtick was typed instead of `~` in `cd ~`, which opens a command substitution | Press Ctrl+C and retype the line |
| `wget -0` didn't save the file | The option is a capital letter O (`-O`), not zero | Use `wget -O uos-installer "<link>"` |
| Wrong installer downloaded | Copied the arm64 link | Use the Linux x64 link |
| `error: unexpected argument 'install' found` | This installer version takes no arguments | Run `sudo ./uos-installer` on its own |
| VM came up on 192.168.20.103 | Typo in the netplan `addresses` line put the server inside the Kea DHCP pool | Fixed the address to `192.168.20.14/24` from the Proxmox console (not SSH, which drops when the IP changes) |
| `sed: unterminated 's' command` | Part of the sed expression was lost while typing it into the noVNC console | Split it into two short sed commands, or edit the file with `nano` |
| UI.com sign-in failed during setup | Possibly DNS (Pi-hole blocking a Ubiquiti endpoint, or the container using its own DNS) | Continued with a local account. Revisit only if remote access is wanted |
| No automatic backup option under Control Plane > Backups | Scheduled backups are tied to a UI.com account | Rely on the Proxmox backup plus manual downloads |

Known issue: some users report UniFi OS Server getting stuck at 100% CPU every few days. If that happens, `uosserver stop && uosserver start` on the VM clears it.

---

## Next Steps
- [ ] Download a UniFi backup file and store it on the PC
- [ ] Delete the `pre-unifi` snapshot once the server is stable (a snapshot is not a backup)
- [ ] Optional: look into the UI.com sign-in error if remote management is wanted
