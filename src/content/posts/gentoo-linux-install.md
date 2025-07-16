---
title: Gentoo Guide 
pubDate: '2025-07-15'
---

[![maxresdefault.jpg](https://i.postimg.cc/rpfpX2cN/maxresdefault.jpg)](https://postimg.cc/p9nvz7mm)

Gentoo is real GNU/Linux. No scripts, no genkernel, no bloat. Just full control. This guide shows how to install Gentoo with UEFI, systemd, and a fully compiled kernel. All manual.

I’m doing this because Gentoo shits on Arch. Arch is everywhere in India now and it’s fucking overrated. Same packages, same configs, just a wiki and vibes. Gentoo actually makes you learn and build your own thing.

---

## 1. Boot the ISO
Download the latest minimal Gentoo ISO and boot it. Select gentoo amd64.

Login as root.

---

## 2. Internet
For Ethernet, it should already work.
For Wi-Fi (Intel cards):

```bash
iwctl
device list
station wlan0 scan
station wlan0 get-networks
station wlan0 connect YourSSID
exit
```

Check:

```bash
ping gentoo.org
```

---

## 3. Partition
Example using `/dev/sda`. Use `fdisk` or `cgdisk`.

### Partitions:

- `/dev/sda1` — EFI System — 512M — EF00
- `/dev/sda2` — Swap — 2G — 8200
- `/dev/sda3` — Root — rest — 8300

### Format:

```bash
mkfs.fat -F32 /dev/sda1
mkswap /dev/sda2
swapon /dev/sda2
mkfs.ext4 /dev/sda3
```

### Mount:

```bash
mount /dev/sda3 /mnt/gentoo
mkdir /mnt/gentoo/boot
mount /dev/sda1 /mnt/gentoo/boot
```

---

## 4. Stage3
Enter the root:

```bash
cd /mnt/gentoo
```

Get the tarball:

```bash
wget https://bouncer.gentoo.org/fetch/root/all/releases/amd64/autobuilds/current-stage3-amd64-systemd/stage3-*.tar.xz
tar xpvf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner
```

---

## 5. Chroot

```bash
cp --dereference /etc/resolv.conf /mnt/gentoo/etc/
mount -t proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev
mount --bind /run /mnt/gentoo/run
chroot /mnt/gentoo /bin/bash
source /etc/profile
export PS1="(chroot) # "
```

---

## 6. Compiler Flags
Edit:

```bash
nano /etc/portage/make.conf
```

Add:

```ini
CFLAGS="-march=native -O2 -pipe"
CXXFLAGS="${CFLAGS}"
MAKEOPTS="-j$(nproc)"
```

---

## 7. Portage Sync

```bash
emerge-webrsync
emerge --sync
```

---

## 8. Profile

```bash
eselect profile list
eselect profile set default/linux/amd64/17.1/systemd
```

---

## 9. Timezone and Locale

```bash
echo "Asia/Kolkata" > /etc/timezone
emerge --config sys-libs/timezone-data

nano /etc/locale.gen
# uncomment en_US.UTF-8
locale-gen

eselect locale list
eselect locale set en_US.utf8
env-update && source /etc/profile
```

---

## 10. Kernel Source

```bash
emerge sys-kernel/gentoo-sources

eselect kernel list
eselect kernel set 1
```

---

## 11. Manual Kernel Compile

```bash
cd /usr/src/linux
make menuconfig
```

### Important options:

- EFI stub support under Processor
- ext4 under File Systems
- Device Drivers > Network card drivers (Intel, Realtek, etc.)
- DEVTMPFS, TMPFS
- Built-in support (not modules) for SATA, NVMe, USB, Filesystem
- Built-in firmware support
- Support for initramfs

Then compile:

```bash
make -j$(nproc)
make modules_install
make install
```

Boot files go to `/boot` automatically.

---

## 12. Fstab
Edit:

```bash
nano /etc/fstab
```

Example:

```bash
/dev/sda1   /boot       vfat    defaults        0 2
/dev/sda2   none        swap    sw              0 0
/dev/sda3   /           ext4    noatime         0 1
```

---

## 13. Hostname and Password

```bash
echo "gentoo" > /etc/hostname
passwd
```

---

## 14. Networking

```bash
emerge dhcpcd
systemctl enable dhcpcd
```

For Wi-Fi:

```bash
emerge iwd
systemctl enable iwd
```

---

## 15. Enable Services

```bash
systemctl preset-all
```

---

## 16. GRUB EFI Bootloader

```bash
emerge grub efibootmgr dosfstools

grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=gentoo
grub-mkconfig -o /boot/grub/grub.cfg
```

---

## 17. Finish

```bash
exit
umount -l /mnt/gentoo/dev{/shm,/pts,}
umount -R /mnt/gentoo
reboot
```

Remove the ISO.

You now have a bootable Gentoo system.

Login as root. Add user:

```bash
useradd -m -G wheel yourname
passwd yourname
emerge sudo
nano /etc/sudoers
# uncomment %wheel ALL=(ALL) ALL
```
---

## Advanced Steps

### 18. Harden the Kernel

To improve security, you can harden the kernel by enabling specific options during `make menuconfig`:

- Disable unused drivers and features to reduce attack surface.
- Enable SELinux or AppArmor for mandatory access control.
- Use grsecurity patches for enhanced security (requires a subscription).
- Enable stack protection and address space layout randomization (ASLR).

Example:

```bash
cd /usr/src/linux
make menuconfig
# Enable SELinux under Security options
# Disable unused drivers under Device Drivers
make -j$(nproc)
make modules_install
make install
```

---

### 19. Install Additional Tools

For better system management and monitoring:

```bash
emerge htop
emerge iotop
emerge sysstat
emerge fail2ban
```

---

### 20. Configure Firewall

Set up `iptables` or `nftables` for network security:

```bash
emerge iptables
systemctl enable iptables
systemctl start iptables

# Example rules
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -j DROP
```

---

### 21. Optimize Boot Time

Use `systemd-analyze` to identify and optimize slow services:

```bash
systemd-analyze blame
systemd-analyze critical-chain
```
---
_Done, khatam. I’ll make another blog soon on how to harden the kernel properly._
