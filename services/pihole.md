# Pi-hole

Pi-hole provides network-wide DNS filtering and ad blocking. It runs as a lightweight LXC container on the Proxmox host and forwards allowed queries to Unbound on OPNsense.

---

## Container Configuration
| Setting | Value |
|---|---|
| Proxmox Host | Dell PowerEdge T340 (`pve`, 192.168.20.11) |
| Container ID | 100 |
| Hostname | pihole |
| Template | `debian-13-standard_13.1-2_amd64` |
| Type | Unprivileged LXC |
| Features | Nesting enabled |
| CPU | 1 core |
| Memory | 512 MB RAM, 512 MB swap |
| Disk | 8 GB on `local-lvm` |
| Network | `vmbr0`, no VLAN tag (lands on Trusted VLAN 20) |
| IP Address | 192.168.20.12/24 (static, set in Proxmox) |
| Gateway | 192.168.20.1 |
| Start at Boot | Yes |
| Web UI | http://192.168.20.12/admin |

The IP sits outside the VLAN 20 Kea DHCP pool (`192.168.20.100` to `.200`), so it can never conflict with a DHCP lease.

---

## DNS Flow
```
Client (VLAN 20)
   │  DNS query
   ▼
Pi-hole (192.168.20.12)  ── blocked domain? ──> returns 0.0.0.0
   │  allowed query
   ▼
Unbound on OPNsense (192.168.20.1)
   │
   ▼
Internet root / upstream servers
```

Pointing Pi-hole's upstream at OPNsense's Unbound (instead of a public resolver like Quad9) keeps local hostname resolution working and keeps all outbound DNS going through the firewall.

---

## Installation Steps
1. Downloaded the `debian-13-standard` template under **local (pve) > CT Templates > Templates**
2. Created the container with **Create CT** using the settings above, leaving **Unprivileged container** checked
3. Enabled **Nesting** under **Options > Features** (see Troubleshooting)
4. Set **Start at boot** to Yes under **Options**
5. Updated the container from its console:
   ```
   apt update
   apt upgrade -y
   apt install -y curl
   reboot
   ```
6. Ran the Pi-hole installer:
   ```
   curl -sSL https://install.pi-hole.net | bash
   ```
7. Installer choices:
   - Static IP warning: **Continue** (the IP is already static in Proxmox)
   - Upstream DNS: **Custom**, `192.168.20.1`
   - Blocklist: default
   - Web interface: installed
   - FTL privacy mode: **0 (Show everything)**, so queries can be traced back to devices when troubleshooting
8. Set the admin password:
   ```
   pihole setpassword
   ```

---

## Verification
From a Windows PC on VLAN 20:

```
C:\> nslookup google.com 192.168.20.12
Server:  pi.hole
Address:  192.168.20.12

Non-authoritative answer:
Name:    google.com
Addresses:  2607:f8b0:4023:100b::64
          ...
          142.250.114.102
          ...
```

`Server: pi.hole` confirms the answer came from the container.

---

## OPNsense DHCP Change (VLAN 20)
To hand out Pi-hole as the DNS server to Trusted devices:
1. **Services > Kea DHCP > Kea DHCPv4 > Subnets**, edit the VLAN 20 subnet
2. Uncheck **Auto collect option data** (otherwise OPNsense fills in its own IP as DNS)
3. Set **DNS servers** to `192.168.20.12`
4. Confirm the router/gateway is still `192.168.20.1`, then Save and Apply
5. On a client, run `ipconfig /release`, `ipconfig /renew`, and `ipconfig /all`, and confirm the DNS server is `192.168.20.12`

Rollback: recheck **Auto collect option data** on the subnet and apply.

---

## Current Scope: VLAN 20 Only
For now, only the Trusted VLAN (20) uses Pi-hole. VLAN 20 is all wired, and devices there reach Pi-hole directly on the same subnet, so no firewall rules are needed.

The other VLANs keep using OPNsense's Unbound for now. Expanding Pi-hole to them is deferred until the wireless side is finalized, because:
- The wireless access point's VLAN setup isn't finished yet
- Wireless clients like the Xbox rely heavily on IPv6. OPNsense can hand out its own DNS server through IPv6 router advertisements, which lets clients bypass Pi-hole unless that is changed as well

Adding another VLAN later takes two changes:
1. A firewall pass rule on that VLAN: TCP/UDP from the VLAN net to a `PiHole` alias (192.168.20.12) on port 53, placed above the block rules
2. The Kea DNS server change above on that VLAN's subnet

---

## Troubleshooting
Problems I hit during setup and how they were fixed:

| Problem | Cause | Fix |
|---|---|---|
| `WARN: Systemd 257 detected. You may need to enable nesting.` when creating the container | Debian 13's systemd needs nesting inside unprivileged containers | Shut down the container, enable **Options > Features > Nesting**, start it again |
| Pasted commands failed with `^[[200~apt: command not found` | The default noVNC console garbles pasted text | Switch the console to **xterm.js** (dropdown next to Console), or type commands by hand |
| Installer failed with `syntax error near unexpected token '<'` and `302 Found` | Typed `-sSl` instead of `-sSL`. Without capital `L`, curl doesn't follow the redirect and downloads an HTML page instead of the script | Rerun with `curl -sSL` |
| `Could not execute systemctl` near the end of `apt upgrade` | systemd itself was upgraded in the running container | Reboot the container before continuing |
| Installer quit with `Installer exited at static IP message` | Pressed Esc or Exit on the static IP warning (the noVNC console draws these menus poorly) | Rerun in the xterm.js console and choose **Continue**. Use Tab/arrows to move, Enter to select, never Esc |

---

## Why an LXC Container Instead of a VM
- Pi-hole is a small single-purpose service, and a container uses a fraction of the RAM and disk a full VM would need
- Containers boot in seconds, so DNS comes back quickly after a host reboot
- As an unprivileged container, root inside Pi-hole maps to an unprivileged user on the Proxmox host, so a compromise of the container does not give admin access to Proxmox

---

## Next Steps
- [ ] Include the container in a scheduled Proxmox backup job
- [ ] Extend Pi-hole to VLANs 10 and 30 once the wireless setup is finished
- [ ] Handle IPv6 DNS (router advertisements) so clients can't bypass Pi-hole
- [ ] Decide on a fallback DNS plan, since DNS on VLAN 20 goes down whenever the T340 or the container is rebooted
