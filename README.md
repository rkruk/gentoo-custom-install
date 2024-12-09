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
mkdir -p /mnt/gentoo/{boot,home,efi}
mount /dev/nvme0n1p1 /mnt/gentoo/efi
mount /dev/nvme0n1p3 /mnt/gentoo/home
mount /dev/sda1 /mnt/gentoo/home/zohak/steam
mount /dev/sdc1 /mnt/gentoo/home/zohak/data
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
USE="nvidia alsa pulseaudio btrfs systemd kde"
VIDEO_CARDS="nvidia"
```

13. Prepare Chroot Environment<br>

```bash
cp -L /etc/resolv.conf /mnt/gentoo/etc/
mount --types proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys && mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev && mount --make-rslave /mnt/gentoo/dev
chroot /mnt/gentoo /bin/bash
source /etc/profile
```

14. Kernel and Bootloader<br><br>
Install Binary Kernel

```bash
emerge --sync
emerge -av sys-kernel/gentoo-kernel-bin
```

15. Configure Fstab Edit `/etc/fstab`:

```bash
/dev/nvme0n1p1  /efi        vfat    defaults,noatime  0 2
/dev/nvme0n1p2  /           btrfs   noatime,compress=zstd  0 1
/dev/nvme0n1p3  /home       btrfs   noatime,compress=zstd  0 2
/dev/sda1       /home/zohak/steam btrfs noatime,compress=zstd 0 2
/dev/sdc1       /home/zohak/data  ext4 defaults,noatime 0 2
/dev/nvme0n1p4  none        swap    sw                   0 0
```

16. Install GRUB:<br><br>

```bash
emerge -av sys-boot/grub
grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=gentoo --recheck
grub-mkconfig -o /boot/grub/grub.cfg
```

17. Post-Installation:<br><br>

Install KDE Plasma

```bash
emerge plasma-meta kde-apps/okular kde-apps/dolphin kde-plasma/plasma-nm
systemctl enable sddm NetworkManager
```

18. Set Optimizations

```bash
echo "vm.swappiness=10" >> /etc/sysctl.conf
echo "vm.vfs_cache_pressure=50" >> /etc/sysctl.conf
```
<br><br>
