<h4>Gentoo Installation Guide with KDE5 and Optimizations</h4>
<br><br>
Here is a detailed step-by-step guide to installing Gentoo with the binary kernel on specific hardware (nvidia with proprietary drivers, asus stx, nvme M2 drive, ssd sata drives, etc...),
taking into account the specific disk configuration, UEFI boot, and systemd setup.<br><br>

1. Boot from the Gentoo installation medium (Minimal Installation ISO) using the UEFI boot option. <br>
2. Select the UEFI boot option (important for systemd setup and EFI partitioning). <br>

Verify you're in UEFI mode:<br>

```
ls /sys/firmware/efi
```
<br>
If the directory exists, you’re in UEFI mode. <br>

Ensure your network connection is working. For wired connections:<br>

```
ping -c 3 google.com
```
<br>
If the network isn’t working, load the driver manually:

```
modprobe r8169
```
<br><br>
3. Disk Partitioning <br>
With the use of he following disks:<br>

1TB NVMe for the root, home, and swap partitions (Btrfs).<br>
2 x 256GB SSDs for Steam library (Btrfs).<br>
512GB SSD for data (a stable file system like ext4).<br>
<br>
Partition the NVMe Disk (1TB) <br><br>
We'll set up:<br><br>
/boot (EFI)<br>
/ <br>
/home <br>
swap <br><br>
on the NVMe drive using Btrfs. <br>
<br>
List your drives:<br>

```
lsblk
```
<br>
Identify your NVMe drive (e.g., /dev/nvme0n1).<br>
<br>
Use gdisk to create the partitions:

```
gdisk /dev/nvme0n1
```
Partition 1: EFI (300MB, type EF00)<br>
Partition 2: Root / (500GB, type 8300)<br>
Partition 3: Home /home (475GB, type 8300)<br>
Partition 4: Swap (16GB, type 8200)<br><br>
Start gdisk to partition the NVMe disk:<br>

```
fdisk /dev/nvme0n1
```
<br>
Create the EFI Partition (300MB, Type EF00)
1. Type n to create a new partition.
2. For Partition Number, press Enter to use the default (1).
3. For First sector, press Enter to accept the default (it will usually start from 2048).
4. For Last sector, type +300M to allocate 300MB for the EFI partition and press Enter.
5. For Hex code or GUID, type EF00 and press Enter (this is the EFI System Partition type).<br>

Create the Root Partition (500GB, Type 8300)
1. Type n to create another partition.
2. For Partition Number, press Enter to use the default (2).
3. For First sector, press Enter to accept the default (it will start right after the EFI partition).
4. For Last sector, type +500G and press Enter.
5. For Hex code or GUID, press Enter to accept the default (8300 for Linux filesystem).<br>

Create the Home Partition (475GB, Type 8300)
1. Type n to create another partition.
2. For Partition Number, press Enter to use the default (3).
3. For First sector, press Enter to accept the default.
4. For Last sector, type +475G and press Enter.
5. For Hex code or GUID, press Enter to accept the default (8300 for Linux filesystem).<br>

Create the Swap Partition (16GB, Type 8200)
1. Type n to create the final partition.
2. For Partition Number, press Enter to use the default (4).
3. For First sector, press Enter to accept the default.
4. For Last sector, type +16G and press Enter.
5. For Hex code or GUID, type 8200 and press Enter (this is the Linux swap partition type).<br>

Write the Partition Table <br>
Type p to print the partition table and verify it looks correct.<br>
If everything looks good, type w to write the partition table and exit gdisk.<br>
Final Partition Table:<br>

```Partition	Type	Size	Hex Code
/dev/nvme0n1p1	EFI	300MB	EF00
/dev/nvme0n1p2	Root (/)	500GB	8300
/dev/nvme0n1p3	Home (/home)	475GB	8300
/dev/nvme0n1p4	Swap	16GB	8200Step 2: Create the EFI Partition (300MB, Type EF00)
```

4. Format the Partitions<br><br>

Format the EFI partition:
```
mkfs.vfat -F32 /dev/nvme0n1p1
```
<br>
Format the root (/) and /home partitions with Btrfs:

```
mkfs.btrfs -f /dev/nvme0n1p2  # root

mkfs.btrfs -f /dev/nvme0n1p3  # home
```
<br>
Format the swap partition:

```
mkswap /dev/nvme0n1p4

swapon /dev/nvme0n1p4
```
<br>
5. Partition the 2 x 256GB SATA SSDs for Steam (Btrfs)<br>
Combine the two 256GB SATA SSDs using Btrfs into a single partition for Steam:

```
gdisk /dev/sda
# 1 partition covering the entire disk (type 8300)
gdisk /dev/sdb
# 1 partition covering the entire disk (type 8300)
```

```
mkfs.btrfs -f -m raid0 -d raid0 /dev/sda /dev/sdb
```
<br>
6. Partition the 512GB SATA SSD (Stable Filesystem)<br>
For the 512GB SSD dedicated to data, we'll use ext4 for stability:

```
gdisk /dev/sdc
# 1 partition covering the entire disk (type 8300)
```

```
mkfs.ext4 /dev/sdc1
```
<br>
7. Mount the Partitions:

```
mount /dev/nvme0n1p2 /mnt/gentoo

mkdir /mnt/gentoo/boot /mnt/gentoo/home /mnt/gentoo/efi

mount /dev/nvme0n1p1 /mnt/gentoo/efi

mount /dev/nvme0n1p3 /mnt/gentoo/home

mkdir /mnt/gentoo/home/zohak

mount /dev/sda1 /mnt/gentoo/home/zohak/steam

mount /dev/sdc1 /mnt/gentoo/home/zohak/data
```
<br>
8. Install the Stage 3 Tarball
Download the systemd stage 3 tarball:

```
cd /mnt/gentoo

wget https://bouncer.gentoo.org/fetch/root/all/releases/amd64/autobuilds/current-stage3-amd64-systemd/stage3-amd64-systemd-*.tar.xz
```
<br>
Extract the tarball:

```
tar xpvf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner
```
<br>
9. Configure the System <br>

Edit /mnt/gentoo/etc/portage/make.conf:

```
nano /mnt/gentoo/etc/portage/make.conf
```
Set CFLAGS, CXXFLAGS, and MAKEOPTS for your AMD Ryzen 5900X:

```
CFLAGS="-march=znver3 -O2 -pipe"
CXXFLAGS="${CFLAGS}"
MAKEOPTS="-j24"
USE="nvidia alsa pulseaudio btrfs systemd kde"
GRUB_PLATFORMS="efi-64"
VIDEO_CARDS="nvidia"
ACCEPT_LICENSE="*"
```

Copy DNS info:

```
cp -L /etc/resolv.conf /mnt/gentoo/etc/
```
<br>
Mount necessary filesystems:

```
mount --types proc /proc /mnt/gentoo/proc

mount --rbind /sys /mnt/gentoo/sys

mount --make-rslave /mnt/gentoo/sys

mount --rbind /dev /mnt/gentoo/dev

mount --make-rslave /mnt/gentoo/dev

mount --rbind /run /mnt/gentoo/run

mount --make-rslave /mnt/gentoo/run
```
<br>
Chroot into the environment:

```
chroot /mnt/gentoo /bin/bash

source /etc/profile

export PS1="(chroot) $PS1"
```
<br>
10. Install the Binary Kernel <br>
Update the Portage tree:

```
emerge --sync
```

Update the system:
```
emerge --sync

emerge -uDU @world
```

Install the Gentoo binary kernel:

```
emerge -av sys-kernel/gentoo-kernel-bin
```

Verify the kernel is installed correctly:

```
eselect kernel list
```

Configure Fstab

Edit /etc/fstab:

```
nano /etc/fstab
```

The fstab configuration:

```
/dev/nvme0n1p1  /efi        vfat    defaults,noatime  0 2
/dev/nvme0n1p2  /           btrfs   noatime,compress=zstd,space_cache=v2,discard=async  0 1
/dev/nvme0n1p3  /home       btrfs   noatime,compress=zstd,space_cache=v2,discard=async  0 2
/dev/sda1       /home/zohak/steam btrfs noatime,compress=zstd,space_cache=v2,discard=async 0 2
/dev/sdc1       /home/zohak/data  ext4   defaults,noatime  0 2
/dev/nvme0n1p4  none        swap    sw                   0 0
```

Install EFI stub support for the kernel:

```
emerge -av sys-boot/efibootmgr
```
<br>
12. Configure the Bootloader (GRUB) <br>
Install GRUB with EFI support:

```
emerge -av sys-boot/grub
```

Install GRUB to the EFI system partition:

```
## grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=gentoo --recheck ## to check
grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=gentoo --recheck
```

Generate the GRUB configuration:

```
grub-mkconfig -o /boot/grub/grub.cfg
```
<br>
Use efibootmgr to check and verify your EFI entries:

```
efibootmgr
```
<br>
<br>
System Configuration <br>
Set hostname and locale:

```
echo "hostname=zohak-gentoo" > /etc/conf.d/hostname

nano /etc/locale.gen  # Uncomment your locales

locale-gen
```
<br>
Set timezone:

```
echo "Europe/Warsaw" > /etc/timezone

emerge --config sys-libs/timezone-data
```
<br>
13. Finish Installation <br>
Set the root password:

```
passwd
```
<br>
Create the user zohak:

```
useradd -m -G users,wheel,audio,video,portage -s /bin/bash zohak

passwd zohak
```
<br>

Exit the chroot:

```
exit
```

Unmount all partitions:

```
umount -l /mnt/gentoo/dev{/shm,/pts,}

umount -l /mnt/gentoo{/proc,/sys,}

umount -R /mnt/gentoo
```
<br>
Reboot the system:

```
reboot
```
<br>

The system should now boot with the Gentoo binary kernel, and the NVIDIA drivers, Btrfs, and all specified hardware should work properly.<br><br>

Post-reboot: Installing KDE5 <br>
After rebooting into your Gentoo system, install KDE Plasma:

```
emerge plasma-meta kde-apps/okular kde-apps/dolphin kde-plasma/plasma-nm
```
<br>
Enable the display manager and NetworkManager:

```
systemctl enable sddm

systemctl enable NetworkManager
```
<br>
NVIDIA Driver Setup<br>
Ensure NVIDIA proprietary drivers are installed and working:

```
emerge -av nvidia-drivers
```
Enable kernel modules:

```
modprobe nvidia
```
<br>
19. Optimization and Scrubbing Setup <br>
Finally, set up monthly Btrfs scrubbing:

```
echo "0 0 1 * * /usr/local/bin/btrfs-scrub.sh" | crontab -
```

Optimizing the Swap Partition <br>
Since the system have installed the 32GB of RAM, swap usage should be minimal. To can optimize swap settings tweak the swappiness and vfs_cache_pressure:
<br>
Set swappiness to a low value (e.g., 10), which tells the system to avoid swapping unless absolutely necessary:

```
echo "vm.swappiness=10" >> /etc/sysctl.conf
```
<br>
Reduce VFS cache pressure (to prioritize caching over frequent inode and dentry reclaiming):

```
echo "vm.vfs_cache_pressure=50" >> /etc/sysctl.conf
```
<br>
3. Systemd and Boot Performance Optimization <br>
To speed up boot times and manage resources better:
<br>
Enable parallel booting in systemd:

```
systemctl enable systemd-readahead-collect systemd-readahead-replay
```
<br>
Disable unneeded services (use systemctl to analyze services you don’t need at boot):

```
systemctl disable <service>
```
<br>
NVIDIA driver optimization: <br>

Make sure to enable "PowerMizer" and use the appropriate performance level by creating a config in /etc/modprobe.d/nvidia.conf:

```
options nvidia NVreg_RegistryDwords="PowerMizerEnable=0x1; PerfLevelSrc=0x2222"
```
<br>
<br>
EFI Partition and Bootloader Stability <br>
To avoid errors with the EFI partition in future kernel upgrades:
<br>
Ensure vfat and EFI options are properly built into the kernel (which should be the case with the binary kernel). Double-check that the kernel is correctly recognizing the partition by running:

```
lsblk -f
```
before and after the upgrade to ensure the /boot/efi mount is functional.
<br>
You can also back up your EFI settings by copying the contents of the /boot/efi directory to a secure location after each successful kernel installation.
<br>
NVIDIA Driver & Kernel Handling <br>
Since you're using proprietary NVIDIA drivers, they can break during kernel upgrades. To avoid this:
<br>
Enable the nvidia use flag in your make.conf (as already mentioned), but also consider: <br>
Using eselect to manage kernel modules automatically. <br>
Install sys-kernel/dkms for automatic recompilation of the NVIDIA driver when the kernel is updated:

```
emerge -av sys-kernel/dkms
```
<br>
Btrfs Maintenance <br>
Since you are using Btrfs, regular maintenance can help prevent issues:
<br>
Enable periodic scrubbing for Btrfs to detect potential file system errors:

```
btrfs scrub start /home
```
<br>
You can automate this with a cron job to run monthly.
<br>
Balance Btrfs partitions occasionally, especially if the disk usage pattern changes frequently:

```
btrfs balance start -dusage=50 /home
```
<br>
8. Kernel Upgrade Process <br>
Since you had issues with the kernel upgrade process, here's a more structured approach:
<br>
Backup /boot and kernel modules before upgrading:

```
cp -r /boot /boot.bak

cp -r /lib/modules/$(uname -r) /lib/modules.bak
```
<br>
After the kernel upgrade, verify that the /boot/efi partition is mounted and update GRUB:

```
mount /boot/efi

grub-mkconfig -o /boot/grub/grub.cfg
```
If you're using the binary kernel, simply use:

```
emerge -av @kernel
```
<br>
By incorporating these optimizations, the system will be more efficient, stable, and performant.
<br><br><br>

To automate Btrfs scrubbing on a monthly basis, you can set up a cron job. Cron is a time-based job scheduler in Linux that allows you to automate tasks at specific intervals.<br>

Here’s how you can set up a cron job for monthly Btrfs scrubbing: 
<br>
Step 1: Create a Script for Btrfs Scrubbing <br>
You can create a simple script to handle the scrubbing. This script will initiate a scrub for your Btrfs partitions (e.g., / and /home) and log the output for review.
<br>
Create the script at /usr/local/bin/btrfs-scrub.sh:

```
sudo nano /usr/local/bin/btrfs-scrub.sh
```
Add the following content to the script:

```
#!/bin/bash

# Scrub the root (/) partition
btrfs scrub start -B / >> /var/log/btrfs-scrub.log 2>&1

# Scrub the /home partition
btrfs scrub start -B /home >> /var/log/btrfs-scrub.log 2>&1

# Scrub the /home/user/steam partition
btrfs scrub start -B /home/user/steam >> /var/log/btrfs-scrub.log 2>&1

# Optional: Scrub any other Btrfs partitions you have
# Example for an additional data partition:
# btrfs scrub start -B /home/user/data >> /var/log/btrfs-scrub.log 2>&1
```
This script will start the scrub process for your Btrfs partitions and log the output to /var/log/btrfs-scrub.log.
<br>
Make the script executable:

```
sudo chmod +x /usr/local/bin/btrfs-scrub.sh
```
<br>
Step 2: Set Up a Cron Job <br>
Now, you can create a cron job that runs this script monthly.
<br>
Open the crontab for the root user:

```
sudo crontab -e
```
Add the following line to schedule the scrub job to run monthly on the 1st day at 10PM:

```
0 22 1 * * /usr/local/bin/btrfs-scrub.sh
```
The schedule here (0 22 1 * *) means "run at 22:00 on the 1st of every month." <br>
Save and exit the crontab.
<br>
Step 3: Verify the Cron Job <br>
You can verify that the cron job is correctly scheduled by listing the cron jobs:

```
sudo crontab -l
```
<br>
Optional: Review Scrub Logs <br>
The output of the scrub process is logged to /var/log/btrfs-scrub.log. You can review this log at any time:

```
cat /var/log/btrfs-scrub.log
```

This setup will ensure that your Btrfs partitions are automatically scrubbed monthly to detect and fix potential file system issues.








