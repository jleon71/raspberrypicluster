# Phase 1: Rocky Linux Installation on Raspberry Pi 4

## Overview
This guide covers installing Rocky Linux 9 (aarch64) on Raspberry Pi 4 with optimal configuration for running K3s.

---

## What You'll Do

1. Download Rocky Linux image for Raspberry Pi
2. Write image to microSD card
3. Initial boot and configuration
4. Set up SSH access
5. Configure network and hostname
6. Prepare system for K3s

**Estimated Time:** 1-2 hours (mostly waiting for downloads)

---

## Prerequisites

### Required Hardware
- Raspberry Pi 4 (4GB+ RAM, 8GB recommended)
- Raspberry Pi 3B+ or newer (ARM64/aarch64 required)
- microSD card reader
- microSD card 32GB+ (64GB+ recommended)
- USB-C power supply (2.5A+)
- Ethernet cable or WiFi

### Required Software on Your Control Machine
- [Balena Etcher](https://www.balena.io/etcher/) (free) - for writing SD card
- SSH client (built into Linux/Mac, built into Windows 10+)
- Or: `dd` command if you're comfortable with Linux

### Important Notes
- Rocky Linux requires ARM64 (aarch64) architecture
- Works on Pi 3B+, 4, 400, 5, and 500
- Does NOT work on Pi 1 or 2 (32-bit only)

---

## Step 1: Download Rocky Linux Image

1. Visit [Rocky Linux Downloads](https://rockylinux.org/download/)
2. Look for **Raspberry Pi Images** section
3. Download **Rocky Linux 9 (aarch64)** for Raspberry Pi
   - Look for: `Rocky-9-Raspberry-Pi-Ostree.aarch64.img.xz` or similar
   - Size: ~400-600MB

**Direct Download Options:**
```bash
# Option 1: Official Rocky Linux repository
wget https://dl.rockylinux.org/pub/rocky/9/images/aarch64/Rocky-9-Raspberry-Pi.img.gz

# Option 2: Alternative mirror
# Check: https://rockylinux.org/download/ for latest version

# After download, extract
gunzip Rocky-9-Raspberry-Pi.img.gz
# or
xz -d Rocky-9-Raspberry-Pi.img.xz
```

**Verify Download:**
```bash
# Check file size
ls -lh Rocky-9-Raspberry-Pi.img*

# Verify integrity (if checksum provided)
sha256sum Rocky-9-Raspberry-Pi.img
```

---

## Step 2: Write Image to microSD Card

### Using Balena Etcher (Recommended - GUI)

1. Insert microSD card into reader
2. Open Balena Etcher
3. Click "Flash from File" → select downloaded `.img.xz` file
4. Select target: your microSD card
5. Click "Flash" and wait (5-10 minutes)
6. Eject card when complete

### Using Command Line (Linux/Mac)

```bash
# Find your SD card device
lsblk    # On Linux
diskutil list  # On macOS

# Decompress image
xz -d debian-bookworm-arm64-rpi.img.xz

# Write to SD card (CAREFUL - replace /dev/sdX with your device!)
# Linux:
sudo dd if=debian-bookworm-arm64-rpi.img of=/dev/sdX bs=4M conv=fsync

# macOS:
sudo dd if=debian-bookworm-arm64-rpi.img of=/dev/rdiskX bs=4m conv=fsync
```

---

## Step 3: First Boot

1. Insert microSD card into Raspberry Pi
2. Connect Ethernet cable (recommended for stability) or have WiFi ready
3. Connect power supply
4. Wait 1-2 minutes for first boot to complete

**First Boot Takes Time:** The Pi will expand the filesystem and initialize. This is normal.

---

## Step 4: SSH Access

### Option A: From Control Machine (if on same network)

```bash
# Find your Pi's IP address
nmap -sn 192.168.1.0/24  # Adjust for your network range

# Or check your router's connected devices

# SSH into the Pi
ssh root@<pi-ip-address>
# Default password varies (may be 'rocky' or empty on first boot)

# Or use the rocky user (default on some builds)
ssh rocky@<pi-ip-address>
```

### Option B: With HDMI Monitor

1. Connect HDMI cable and USB keyboard
2. Login as `root` with password `raspberry`
3. Configure WiFi or network
4. Get IP address: `hostname -I`

---

## Step 5: Initial System Configuration

Run these commands on the Raspberry Pi:

```bash
# Login as root (if not already)
sudo su -

# Update system (dnf for Rocky Linux)
dnf update -y
dnf upgrade -y

# Change hostname (replace "rpi-master" with your choice)
hostnamectl set-hostname rpi-master
sed -i 's/localhost/rpi-master/g' /etc/hosts

# Change root password (IMPORTANT - use strong password)
passwd
# Enter new password twice

# Create a regular user (for security)
useradd -m -G wheel -s /bin/bash rocky
passwd rocky
# Enter password for rocky user

# Configure sudoers for new user (Rocky uses 'wheel' group)
# Check if already configured
grep wheel /etc/sudoers

# Configure static IP (if using Ethernet)
# Edit /etc/network/interfaces or use nmcli (NetworkManager)
```

### Configure Static IP (Recommended - Using nmcli)

Rocky Linux uses NetworkManager by default. Configure with nmcli:

```bash
# List network connections
nmcli connection show

# Create static IP connection
nmcli connection add type ethernet con-name ethernet-static \
  ifname eth0 \
  ip4 192.168.1.100/24 \
  gw4 192.168.1.1

# Set DNS servers
nmcli connection modify ethernet-static \
  ipv4.dns "8.8.8.8 8.8.4.4"

# Activate connection
nmcli connection up ethernet-static

# Verify
ip addr show eth0
```

**Alternative: Edit Network Config File**

```bash
# Edit connection config
sudo nano /etc/NetworkManager/conf.d/99-custom.conf

# Or edit connection file directly
sudo nano /etc/NetworkManager/system-connections/ethernet-static.nmconnection
```

Add:
```
[ipv4]
method=manual
address1=192.168.1.100/24,192.168.1.1
dns=8.8.8.8;8.8.4.4;
```

Restart networking:

```bash
sudo systemctl restart NetworkManager
# or reboot
reboot
```

---

## Step 6: SSH Key Setup (Recommended)

On your **control machine**, set up passwordless SSH:

```bash
# Generate SSH key (if you don't have one)
ssh-keygen -t ed25519 -C "your-email@example.com"

# Copy public key to Raspberry Pi (using 'rocky' user for Rocky Linux)
ssh-copy-id -i ~/.ssh/id_ed25519.pub rocky@<pi-ip>

# Test passwordless login
ssh rocky@<pi-ip>

# Or if root is available
ssh-copy-id -i ~/.ssh/id_ed25519.pub root@<pi-ip>
ssh root@<pi-ip>
```

---

## Step 7: Prepare for K3s

Run these commands on each Raspberry Pi:

```bash
# Switch to debian user (or your regular user)
su - debian

# Enable cgroup v2 (required for K3s)
sudo nano /boot/firmware/cmdline.txt
```

Add at the end of the line: `cgroup_memory=1 cgroup_enable=memory`

**Before:**
```
console=ttyAMA0 root=/dev/mmcblk0p2 rw rootwait
```

**After:**
```
console=ttyAMA0 root=/dev/mmcblk0p2 rw rootwait cgroup_memory=1 cgroup_enable=memory
```

Reboot to apply:
```bash
sudo reboot
```

---

## Step 8: Verify System is Ready

After reboot, verify everything is working:

```bash
# Check system info
uname -a
# Should show: Linux ... aarch64 ... (Rocky Linux)

# Check system release
cat /etc/os-release
# Should show: Rocky Linux 9

# Check memory
free -h
# Should show available RAM

# Check disk
df -h
# Should show mounted filesystems

# Check network
ip addr
# Should show IP address(es)

# Check cgroups (required for K3s)
cat /proc/cmdline | grep cgroup
# Should show: cgroup_memory=1 cgroup_enable=memory

# Verify firewall status
sudo systemctl status firewalld
# Should be: active (running)

# List firewall zones
sudo firewall-cmd --list-all
```

---

## Repeat for Additional Raspberry Pis

For each additional Pi you want in your cluster:

1. Repeat steps 1-7 with different hostnames
   - `rpi-worker-1`, `rpi-worker-2`, etc.
2. Use different static IPs
   - `192.168.1.101`, `192.168.1.102`, etc.

---

## Troubleshooting

### Pi doesn't boot
- Check power supply (needs 2.5A+)
- Try reformatting SD card and re-flashing
- Check HDMI/keyboard if using monitor

### Can't SSH into Pi
- Verify Ethernet is connected and has IP address
- Try `nmap` or check router for connected devices
- Try default login: `root` / `raspberry`

### Network connectivity issues
- Check `/etc/network/interfaces` configuration
- Restart networking: `sudo systemctl restart networking`
- Check cables and router

### cgroup error
- Make sure you edited `/boot/firmware/cmdline.txt` correctly
- Reboot after editing: `sudo reboot`
- Verify with: `cat /proc/cmdline | grep cgroup`

---

## Security Notes

⚠️ **Important:** This is a fresh system. Before going production:

- Change root password
- Create regular user account (rocky or custom)
- Set up SSH keys
- Disable SSH password login (covered in Ansible playbooks)
- Configure firewall with firewalld (also in Ansible playbooks)
- Keep system updated: `sudo dnf update -y && sudo dnf upgrade -y`
- Disable SELinux if needed for easier testing (can be re-enabled later)
  ```bash
  sudo setenforce 0  # Temporarily
  # Or edit /etc/selinux/config for permanent change
  ```

---

## Next Step

Once you have Debian installed and SSH access working on at least one Raspberry Pi, proceed to:
**02_ANSIBLE_SETUP.md** - Set up Ansible for automated configuration management

---

## Quick Reference

| Command | Purpose |
|---------|---------|
| `ssh rocky@<ip>` | Connect to Pi |
| `hostnamectl set-hostname NAME` | Change hostname |
| `passwd` | Change password |
| `sudo reboot` | Restart Pi |
| `sudo systemctl restart NetworkManager` | Restart network |
| `nmcli connection show` | Show network connections |
| `sudo firewall-cmd --list-all` | Show firewall rules |
| `ip addr` | Show IP addresses |
| `df -h` | Show disk usage |
| `free -h` | Show memory usage |
| `dnf update -y` | Update all packages |
| `sudo setenforce 0` | Disable SELinux (temporary) |

---

**Status:** Rocky Linux installation and basic configuration complete
**Next:** Ansible setup and cluster provisioning
