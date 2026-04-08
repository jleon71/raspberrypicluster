# Phase 1: Debian Installation on Raspberry Pi 4

## Overview
This guide covers installing Debian 12 (Bookworm) on Raspberry Pi 4 with optimal configuration for running K3s.

---

## What You'll Do

1. Download Debian image for Raspberry Pi
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
- microSD card reader
- microSD card 32GB+ (64GB+ recommended)
- USB-C power supply (2.5A+)
- Ethernet cable or WiFi

### Required Software on Your Control Machine
- [Balena Etcher](https://www.balena.io/etcher/) (free) - for writing SD card
- SSH client (built into Linux/Mac, built into Windows 10+)
- Or: `dd` command if you're comfortable with Linux

---

## Step 1: Download Debian Image

1. Visit [Raspberry Pi Debian Images](https://www.raspberrypi.com/software/)
2. Download **Debian 12 (Bookworm)** for Raspberry Pi 4
   - Look for: `debian-12-generic-arm64+raspi.img.xz` or similar
   - Size: ~300-500MB

**Alternative Direct Download:**
```bash
# On your control machine
wget https://raspi.debian.net/verified/debian-bookworm-arm64-rpi.img.xz
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
# Default password: raspberry
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

# Update system
apt update && apt upgrade -y

# Change hostname (replace "rpi-master" with your choice)
hostnamectl set-hostname rpi-master
sed -i 's/localhost/rpi-master/g' /etc/hosts

# Change root password (IMPORTANT - use strong password)
passwd
# Enter new password twice

# Create a regular user (for security)
useradd -m -G sudo -s /bin/bash debian
passwd debian
# Enter password for debian user

# Configure static IP (if using Ethernet)
# Edit /etc/network/interfaces or use netplan
```

### Configure Static IP (Recommended)

Edit `/etc/network/interfaces`:

```bash
nano /etc/network/interfaces
```

Add to the file:

```
# Ethernet
auto eth0
iface eth0 inet static
    address 192.168.1.100
    netmask 255.255.255.0
    gateway 192.168.1.1
    dns-nameservers 8.8.8.8 8.8.4.4
```

Restart networking:

```bash
systemctl restart networking
# or reboot
reboot
```

---

## Step 6: SSH Key Setup (Recommended)

On your **control machine**, set up passwordless SSH:

```bash
# Generate SSH key (if you don't have one)
ssh-keygen -t ed25519 -C "your-email@example.com"

# Copy public key to Raspberry Pi
ssh-copy-id -i ~/.ssh/id_ed25519.pub debian@<pi-ip>

# Test passwordless login
ssh debian@<pi-ip>
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
# Should show: Linux arm64 (or aarch64)

# Check memory
free -h
# Should show available RAM

# Check disk
df -h
# Should show mounted filesystems

# Check network
ip addr
# Should show IP address(es)

# Check cgroups
cat /proc/cmdline | grep cgroup
# Should show: cgroup_memory=1 cgroup_enable=memory
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
- Create regular user account
- Set up SSH keys
- Disable SSH password login (covered in Ansible playbooks)
- Configure firewall (also in Ansible playbooks)
- Keep system updated: `sudo apt update && sudo apt upgrade`

---

## Next Step

Once you have Debian installed and SSH access working on at least one Raspberry Pi, proceed to:
**02_ANSIBLE_SETUP.md** - Set up Ansible for automated configuration management

---

## Quick Reference

| Command | Purpose |
|---------|---------|
| `ssh debian@<ip>` | Connect to Pi |
| `hostnamectl set-hostname NAME` | Change hostname |
| `passwd` | Change password |
| `sudo reboot` | Restart Pi |
| `systemctl restart networking` | Restart network |
| `ip addr` | Show IP addresses |
| `df -h` | Show disk usage |
| `free -h` | Show memory usage |

---

**Status:** Debian installation and basic configuration complete
**Next:** Ansible setup and cluster provisioning
