## Gentoo Installation Guide with KDE6 and Optimizations
<br>
This step-by-step guide details the installation of Gentoo with a binary kernel, UEFI boot, systemd setup, and optimizations - focusing on specific hardware configuration:<br> 

  -  ASUS Xonar STX,<br>
  -  NVMe M.2,<br>
  -  SATA SSDs.<br>

### Preparation

1. Boot from Installation Medium<br>
Use the Gentoo Minimal Installation ISO and ensure you boot in UEFI mode.<br>
Select the UEFI boot option in your system's BIOS/UEFI firmware.<br>

2. Verify UEFI Mode

```bash
ls /sys/firmware/efi
```
If the directory exists, you are in UEFI mode.<br>

3. Ensure Network Connectivity<br>
For wired connections:

```bash
ping -c 3 google.com
```
If it fails, manually load the appropriate network driver (e.g., for Realtek NICs):

```bash
modprobe r8169
```

4. Disk Partitioning<br>
Disk Setup (my setup as an example):<br>

```bash
1TB NVMe:
  Root (/),
  Home (/home),
  Swap (Btrfs)
2 x 256GB SSDs:
  Steam Library (Btrfs, RAID 0)
512GB SSD:
  Data (ext4)
```

5. Identify Disks

```bash
lsblk
```

6. Locate your drives (e.g., /dev/nvme0n1 for NVMe).<br>

7. Partition NVMe Drive

```bash
gdisk /dev/nvme0n1
```

Create the following partitions:

```bash
EFI (/dev/nvme0n1p1, 300MB, EF00)
Root (/dev/nvme0n1p2, 500GB, 8300)
Home (/dev/nvme0n1p3, ~475GB, 8300)
Swap (/dev/nvme0n1p4, 16GB, 8200)
```

8. Partition SSDs
For Steam (RAID 0):

```bash
gdisk /dev/sda
gdisk /dev/sdb
mkfs.btrfs -f -m raid0 -d raid0 /dev/sda1 /dev/sdb1
```

9. For data on the 512GB SSD:

```bash
gdisk /dev/sdc
mkfs.ext4 /dev/sdc1
```

10. Formatting and Mounting<br>
Format Partitions

```bash
mkfs.vfat -F32 /dev/nvme0n1p1         # EFI
mkfs.btrfs -f /dev/nvme0n1p2         # Root
mkfs.btrfs -f /dev/nvme0n1p3         # Home
mkswap /dev/nvme0n1p4 && swapon /dev/nvme0n1p4
```

Mount Partitions<br>

```bash
mount /dev/nvme0n1p2 /mnt/gentoo
mkdir /mnt/gentoo/boot /mnt/gentoo/home /mnt/gentoo/efi
```

```bash
mount /dev/nvme0n1p2 /mnt/gentoo
mkdir -p /mnt/gentoo/{boot,home,efi}
# It is the same as the: mkdir /mnt/gentoo/boot /mnt/gentoo/home /mnt/gentoo/efi
mount /dev/nvme0n1p1 /mnt/gentoo/efi
mount /dev/nvme0n1p3 /mnt/gentoo/home
mkdir /mnt/gentoo/home/gentoouser
mount /dev/sda1 /mnt/gentoo/home/gentoouser/steam
mount /dev/sdc1 /mnt/gentoo/home/gentoouser/data
```

11. Stage 3 Installation<br><br>
Download and Extract Stage 3

```bash
cd /mnt/gentoo
wget https://bouncer.gentoo.org/fetch/root/all/releases/amd64/autobuilds/current-stage3-amd64-systemd/stage3-amd64-systemd-*.tar.xz
tar xpvf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner
```

12. Configure Makefile:<br><br> 

Edit `/mnt/gentoo/etc/portage/make.conf:`

```bash
CFLAGS="-march=znver3 -O2 -pipe"
CXXFLAGS="${CFLAGS}"
MAKEOPTS="-j24"
USE="nvidia alsa pulseaudio btrfs systemd kde qt6"
VIDEO_CARDS="nvidia"
GRUB_PLATFORMS="efi-64"
ACCEPT_LICENSE="*"
```

13. Prepare Chroot Environment<br>
<br>
Copy DNS information to the chroot environment:

```bash
cp -L /etc/resolv.conf /mnt/gentoo/etc/
```
<br>
Mount necessary filesystems:

```bash
mount --types proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys && mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev && mount --make-rslave /mnt/gentoo/dev
mount --rbind /run /mnt/gentoo/run && mount --make-rslave /mnt/gentoo/run
```
<br>
Chroot into the Gentoo environment:<br>

```bash
chroot /mnt/gentoo /bin/bash
source /etc/profile
export PS1="(chroot) $PS1"
```

14. Kernel and Bootloader<br><br>
Install Binary Kernel

```bash
emerge --sync && emerge -uDU world
emerge -av sys-kernel/gentoo-kernel-bin
```
<br>
Verify the kernel version:

```bash
eselect kernel list
```
<br>

15. Configure Fstab Edit `/etc/fstab`:

```bash
/dev/nvme0n1p1  /efi        vfat    defaults,noatime  0 2
/dev/nvme0n1p2  /           btrfs   noatime,compress=zstd,space_cache=v2,discard=async  0 1
/dev/nvme0n1p3  /home       btrfs   noatime,compress=zstd,space_cache=v2,discard=async  0 2
/dev/sda1       /home/gentoouser/steam btrfs noatime,compress=zstd,space_cache=v2,discard=async 0 2
/dev/sdc1       /home/gentoouser/data  ext4   defaults,noatime  0 2
/dev/nvme0n1p4  none        swap    sw                   0 0
```
<br>
Install EFI boot manager:

```bash
emerge -av sys-boot/efibootmgr
```
<br>

16. Install GRUB:<br><br>

```bash
emerge -av sys-boot/grub
grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=gentoo --recheck
grub-mkconfig -o /boot/grub/grub.cfg
```
<br>
Check EFI boot entries:

```bash
efibootmgr
```
<br>

17.System Configuration<br><br>

Set the hostname:<br>

```bash
echo "hostname=lovelysystemname-gentoo" > /etc/conf.d/hostname
```
<br>
Edit the locale configuration:

```bash
nano /etc/locale.gen
```
<br>
Uncomment your desired locales and generate them:<br>

```bash
locale-gen
```
<br>
Set the timezone:<br>

```bash
echo "Europe/Warsaw" > /etc/timezone
emerge --config sys-libs/timezone-data
```

Set the root password:<br>

```bash
passwd
```
<br>
Create a user:

```bash
useradd -m -G users,wheel,audio,video,portage -s /bin/bash gentoouser
passwd gentoouser
```
<br>
18. Finish Installation<br><br>
Exit the chroot environment:

```bash
exit
```
<br>
Unmount all partitions:<br>

```bash
umount -l /mnt/gentoo/dev{/shm,/pts,}
umount -l /mnt/gentoo{/proc,/sys,}
umount -R /mnt/gentoo
```
<br>
Reboot the system:<br>

```bash
reboot
```
<br>
19. Post-Reboot: Installing KDE Plasma<br><br>
After rebooting into Gentoo, install KDE Plasma:<br>

```bash
emerge plasma-meta kde-apps/okular kde-apps/dolphin kde-plasma/plasma-nm media-video/pipewire x11-misc/sddm
systemctl enable --now pipewire pipewire-pulse
```
<br>
Enable the display manager and NetworkManager:

```bash
systemctl enable sddm
systemctl enable NetworkManager
```
<br>
20. NVIDIA Driver Setup<br><br>
Install the NVIDIA proprietary drivers:

```bash
emerge -av nvidia-drivers
```
<br>
Enable the NVIDIA kernel modules:<br>

```bash
modprobe nvidia
```
<br>
21. Optimization and Maintenance<br><br>
Enable Btrfs Scrubbing<br>
Create a script for scrubbing Btrfs partitions:<br>

```bash
sudo nano /usr/local/bin/btrfs-scrub.sh
```

Add the following:

```bash
#!/bin/bash
btrfs scrub start -B / >> /var/log/btrfs-scrub.log 2>&1
btrfs scrub start -B /home >> /var/log/btrfs-scrub.log 2>&1
btrfs scrub start -B /home/gentoouser/steam >> /var/log/btrfs-scrub.log 2>&1
```
<br>
Make the script executable:

```bash
sudo chmod +x /usr/local/bin/btrfs-scrub.sh
```
<br>
Set up a cron job for monthly scrubbing:

```bash
sudo crontab -e
```
<br>
Add:

```bash
0 22 1 * * /usr/local/bin/btrfs-scrub.sh
```
<br>

22. Post-Installation:<br><br>

```bash
echo "vm.swappiness=10" >> /etc/sysctl.conf
echo "vm.vfs_cache_pressure=50" >> /etc/sysctl.conf
```
<br><br>
