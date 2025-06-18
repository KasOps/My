
Instruction: migrating from RAID 1 to RAID 10 using LVM and four NVMe disks

# How to Migrate from RAID 1 to RAID 10 without Stopping the Server (Using LVM and Four NVMe Disks)

**Description:**  
This guide covers the process of migrating from RAID 1 to RAID 10 through LVM without stopping the server. Suitable for both full migration to new disks and scenarios involving existing RAID1 disks.

---

## Contents

- [Requirements](#requirements)  
- [Step 1. Preparing GPT and EFI partitions on each disk](#step-1-preparing-gpt-and-efi-partitions-on-each-disk)  
- [Step 2. Creating RAID10 array](#step-2-creating-raid10-array)  
- [Step 3. Connecting the new RAID10 to LVM](#step-3-connecting-the-new-raid10-to-lvm)  
- [Step 4. Moving data from md0 to md1](#step-4-moving-data-from-md0-to-md1)  
- [Step 5. Cleaning superblocks of the old RAID](#step-5-cleaning-superblocks-of-the-old-raid)  
- [Step 6. Updating mdadm configuration](#step-6-updating-mdadm-configuration)  
- [Step 7. Formatting EFI partitions](#step-7-formatting-efi-partitions)  
- [Step 8. Setting up EFI and GRUB](#step-8-setting-up-efi-and-grub)  
- [Step 9. Removing the old RAID1 array](#step-9-removing-the-old-raid1-array)  
- [Recommendations](#recommendations)

---

## Requirements

- Running RAID 1 with LVM  
- Four NVMe disks for RAID 10  
- Installed packages: `mdadm`, `parted`, `lvm2`

---

## Step 1. Preparing GPT and EFI partitions on each disk

On each of the four disks, create:

- EFI partition (1MiB – 1128MiB)  
- Main partition for RAID (1128MiB – 100%)

For the first disk:

```bash
parted /dev/nvme0n1 --script mklabel gpt
parted /dev/nvme0n1 --script mkpart primary fat32 1MiB 1128MiB
parted /dev/nvme0n1 --script set 1 boot on
parted /dev/nvme0n1 --script set 1 esp on
parted /dev/nvme0n1 --script mkpart primary 1128MiB 100%

For the others — use a loop:

```bash
for disk in /dev/nvme1n1 /dev/nvme2n1 /dev/nvme3n1; do
  parted "$disk" --script mklabel gpt
  parted "$disk" --script mkpart primary fat32 1MiB 1128MiB
  parted "$disk" --script set 1 boot on
  parted "$disk" --script set 1 esp on
  parted "$disk" --script mkpart primary 1128MiB 100%
done

Step 2. Creating RAID10 array

Option 1: Using 4 new disks
```bash
mdadm --create /dev/md1 --level=10 --raid-devices=4 \
 /dev/nvme0n1p2 /dev/nvme1n1p2 /dev/nvme2n1p2 /dev/nvme3n1p2

Option 2: Starting with two disks, adding two later
```bash
mdadm --create /dev/md1 --level=10 --raid-devices=4 \
 /dev/nvme0n1p2 missing /dev/nvme1n1p2 missing


Step 3. Connecting the new RAID10 to LVM
```bash
pvcreate /dev/md1
vgextend sys /dev/md1

Step 4. Moving data from md0 to md1
⚠️ Important: This operation is long. Use screen or tmux to avoid losing SSH access if connection drops.

Start screen:
```bash
screen -S pvmove

Inside screen, run:
```bash
pvmove -v /dev/md0 /dev/md1

After completion:
```bash
vgreduce sys /dev/md0
pvremove /dev/md0
To detach from screen without stopping it:
Press Ctrl+A then D

To resume:
```bash
screen -r pvmove


Step 5. Cleaning superblocks of the old RAID
```bash
mdadm --zero-superblock /dev/sda2
mdadm --zero-superblock /dev/sdb2

Step 6. Updating mdadm configuration
```bash
mdadm --detail --scan >> /etc/mdadm/mdadm.conf

Step 7. Formatting EFI partitions
```bash
mkfs.vfat -F32 /dev/sda1
mkfs.vfat -F32 /dev/sdb1
mkfs.vfat -F32 /dev/sdc1
mkfs.vfat -F32 /dev/sdd1

Step 8. Setting up EFI and GRUB

Unmount EFI if mounted:
```bash
umount /boot/efi

Get UUID of the EFI partition:
```bash
blkid /dev/sda1
Example output:
```pgsql
/dev/sda1: UUID="7B1F-36F2" TYPE="vfat"

Update /etc/fstab:
```bash
nano /etc/fstab
⚠️ Important: Replace only the UUID value inside quotes, leave the rest of the line unchanged.

Example:

Before:
```pgsql
UUID=OLD-UUID-HERE /boot/efi vfat umask=0077 0 1
After:
```pgsql
UUID=7B1F-36F2 /boot/efi vfat umask=0077 0 1

Mount and reinstall the bootloader:
```bash
mount /boot/efi
dpkg-reconfigure grub-efi-amd64
update-grub
Repeat for other EFI disks if needed for reliable booting from any.


Step 9. Removing the old RAID1 array
```bash
mdadm --stop /dev/md0
mdadm --zero-superblock /dev/sda2
mdadm --zero-superblock /dev/sdb2



Recommendations:

Use screen or tmux for all long-running operations.

Check RAID status with:
```bash
cat /proc/mdstat

Ensure /boot/efi mounts correctly at boot:
```bash
mount | grep efi




