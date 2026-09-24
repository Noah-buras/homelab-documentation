# Ubuntu Server Template

A reusable Ubuntu Server VM template on Proxmox. New lab machines are cloned from it, so each one starts already updated, hardened for SSH key login, and ready to use in a few minutes.

---

## Template Summary
| Setting | Value |
|---|---|
| VM ID | 200 (template) |
| Name | ubuntu-base |
| OS | Ubuntu Server 26.04 LTS |
| CPU | 2 cores, type `host` |
| Memory | 2048 MB |
| Disk | 32 GB on `local-lvm` (Discard and SSD emulation on) |
| Disk Layout | Single ext4 partition, no LVM |
| Network | `vmbr0`, VirtIO, no VLAN tag (Trusted VLAN 20) |
| Guest Agent | `qemu-guest-agent` installed, Qemu Agent enabled in Proxmox |
| Default User | `noah` (sudo), SSH key login only |

VM IDs 200 and up are used for VMs, and 100 and up for LXC containers, so the two are easy to tell apart.

---

## Building the Base VM

### 1. Get the ISO onto Proxmox
Downloaded straight to the server with **local (pve) > ISO Images > Download from URL**, using the direct file link from `releases.ubuntu.com/26.04/`.

### 2. Create the VM
Created with **Create VM** using the settings above, with **Qemu Agent** checked on the System tab.

### 3. Install Ubuntu Server
Installer choices:
- Install type: **Ubuntu Server** (not minimized)
- Network: DHCP (the template stays on DHCP so clones don't share an IP)
- Storage: **Use an entire disk**, with **Set up this disk as an LVM group** unchecked
- Profile: user `noah`, hostname `ubuntu-base`
- **Install OpenSSH server**: checked
- Featured snaps: none

### 4. Update and install the guest agent
```
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y qemu-guest-agent
sudo systemctl start qemu-guest-agent
```
With the agent running, Proxmox shows the VM's IP on its Summary page and can shut it down and back it up cleanly.

---

## SSH Key Authentication

### 1. Generate a key on the Windows PC (PowerShell)
```
ssh-keygen -t ed25519
```
The private key stays on the PC and is protected with a passphrase. The public key (`id_ed25519.pub`) is what goes on servers.

### 2. Copy the public key to the VM
Windows doesn't include `ssh-copy-id`, so the key is piped over SSH:
```
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh noah@<vm-ip> "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

### 3. Disable password and root login over SSH
```
echo -e "PasswordAuthentication no\nPermitRootLogin no" | sudo tee /etc/ssh/sshd_config.d/10-hardening.conf
sudo systemctl reload ssh
```
SSH reads the files in `sshd_config.d` in alphabetical order and keeps the first value it finds, so the `10-` prefix makes this file win over Ubuntu's `50-cloud-init.conf`.

Verified with:
```
sudo sshd -T | grep -E "passwordauthentication|permitrootlogin"
```

### 4. Test from the PC
| Test | Expected Result |
|---|---|
| `ssh noah@<vm-ip>` | Logs in with only the key passphrase |
| `ssh -o PubkeyAuthentication=no noah@<vm-ip>` | `Permission denied (publickey)` |

The Proxmox Console still accepts the password, since it acts like a local keyboard rather than SSH. That is the fallback if SSH ever breaks.

---

## Preparing the Template
Some IDs have to be unique on every machine, so they are cleared before converting.

```
sudo apt clean

# Reset machine-id so each clone generates its own (Ubuntu uses it as the DHCP client ID)
sudo truncate -s 0 /etc/machine-id
sudo rm -f /var/lib/dbus/machine-id
sudo ln -s /etc/machine-id /var/lib/dbus/machine-id

# Regenerate SSH host keys automatically on each clone's first boot
sudo mkdir -p /etc/systemd/system/ssh.service.d
printf "[Service]\nExecStartPre=\nExecStartPre=/usr/bin/ssh-keygen -A\nExecStartPre=/usr/sbin/sshd -t\n" | sudo tee /etc/systemd/system/ssh.service.d/regen-hostkeys.conf
sudo systemctl daemon-reload
sudo rm -f /etc/ssh/ssh_host_*

sudo shutdown now
```

Then in Proxmox:
1. **Hardware > CD/DVD Drive**: set to **Do not use any media**
2. Right-click VM 200 > **Convert to template**

The VM is not booted again after cleanup, or it would regenerate the IDs that were just cleared.

---

## Cloning a New Lab Machine
1. Right-click the template > **Clone**, **Full Clone**, target storage `local-lvm`
2. Start the clone and log in through the Proxmox Console
3. Set the hostname:
   ```
   sudo hostnamectl set-hostname <name>
   sudo sed -i 's/ubuntu-base/<name>/g' /etc/hosts
   ```
4. Set a static IP by editing `/etc/netplan/00-installer-config.yaml`:
   ```yaml
   network:
     version: 2
     ethernets:
       ens18:
         dhcp4: false
         addresses: [192.168.20.XX/24]
         routes:
           - to: default
             via: 192.168.20.1
         nameservers:
           addresses: [192.168.20.12]
   ```
5. Apply it safely:
   ```
   sudo chmod 600 /etc/netplan/*.yaml
   sudo netplan try
   ```
   `netplan try` rolls the change back after 120 seconds unless Enter is pressed, so a bad config can't cut off access.
6. Test from the PC with `ssh noah@192.168.20.XX` (accept the new host fingerprint)

---

## Clones
| ID | Name | IP | Purpose |
|---|---|---|---|
| 201 | lab-01 | 192.168.20.21 | General-purpose lab machine |

---

## Troubleshooting
| Problem | Cause | Fix |
|---|---|---|
| Proxmox named the download `thank-you` and failed with `wrong file extension` | The link from ubuntu.com's download button points to a thank-you web page, not the ISO | Copy the direct `.iso` link from `releases.ubuntu.com/26.04/` instead |
| (Avoided) Only about half the disk usable after install | Ubuntu's default LVM layout only allocates part of the disk to `/` | Unchecked the LVM option in the installer so `/` uses the full 32 GB |
| Password-refusal test prompted to trust a new host | The test was run from inside the VM, so it was SSH'ing into itself | Run SSH tests from the PC. A `PS C:\` prompt is the PC, `noah@<hostname>` is the VM |
| `netplan try` failed with `unknown key 'version0'` | A typo in the first key of the YAML file (an earlier `versions` typo was caught before running it) | The line must read exactly `version: 2`. Netplan rejects the whole file on any unknown key |
| Prompt still showed `ubuntu-base` after changing the hostname | The prompt only updates on a new login | Confirm with `hostnamectl`, then log out and back in |
