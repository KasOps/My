# How to Migrate from RAID 1 to RAID 10 Without Stopping the Server (Using LVM and Four NVMe Disks)

## Overview

This guide describes the process of migrating from a RAID 1 array to a RAID 10 array using LVM, without shutting down the server. The migration works for setups where you're using four new NVMe disks, or a mix of new and old disks.

---

## Step 1 — Partition All Four NVMe Disks (GPT + EFI)

Create GPT labels and partitions on all four disks. Each disk should have:

- 1st partition: EFI (1MiB – 1128MiB)
- 2nd partition: for RAID (1128MiB – 100%)

For the first disk:

```bash
parted /dev/nvme0n1 --script mklabel gpt
parted /dev/nvme0n1 --script mkpart primary fat32 1MiB 513MiB
parted /dev/nvme0n1 --script set 1 boot on
parted /dev/nvme0n1 --script set 1 esp on
parted /dev/nvme0n1 --script mkpart primary 513MiB 100%
```

Repeat for the remaining disks:

```bash
for disk in /dev/nvme1n1 /dev/nvme2n1 /dev/nvme3n1; do
  parted "$disk" --script mklabel gpt
  parted "$disk" --script mkpart primary fat32 1MiB 513MiB
  parted "$disk" --script set 1 boot on
  parted "$disk" --script set 1 esp on
  parted "$disk" --script mkpart primary 513MiB 100%
done
```

---

## Step 2 — Create the RAID10 Array

### Option A: You have all 4 disks available

```bash
mdadm --create /dev/md1 --level=10 --raid-devices=4 \
  /dev/nvme0n1p2 /dev/nvme1n1p2 /dev/nvme2n1p2 /dev/nvme3n1p2
```

### Option B: You have only 2 disks now, and will add 2 later

```bash
mdadm --create /dev/md1 --level=10 --raid-devices=4 \
  /dev/nvme0n1p2 missing /dev/nvme1n1p2 missing
```

---

## Step 3 — Add RAID10 to LVM

```bash
pvcreate /dev/md1
vgextend sys /dev/md1
```

---

## Step 4 — Move Data from Old RAID1 (md0) to New RAID10 (md1)

**Important**: This is a long operation. Use `screen` or `tmux`.

```bash
screen -S pvmove
pvmove -v /dev/md0 /dev/md1
vgreduce sys /dev/md0
pvremove /dev/md0
```

To detach from screen:

```bash
Ctrl+A then D
```

To reattach:

```bash
screen -r pvmove
```

---

## Step 5 — Clear Superblocks on Old RAID1 Disks

```bash
mdadm --zero-superblock /dev/sda2
mdadm --zero-superblock /dev/sdb2
```

---

## Step 6 — Update mdadm Configuration

```bash
mdadm --detail --scan >> /etc/mdadm/mdadm.conf
```

---

## Step 7 — Add Old Disks to the RAID10 (if applicable)

If you started with two disks and now want to add two more:

```bash
mdadm --add /dev/md1 /dev/nvme2n1p2
mdadm --add /dev/md1 /dev/nvme3n1p2
watch cat /proc/mdstat
mdadm --detail /dev/md1
```

---

## Step 8 — Format EFI Partitions on All Disks

```bash
mkfs.vfat -F32 /dev/nvme0n1p1
mkfs.vfat -F32 /dev/nvme1n1p1
mkfs.vfat -F32 /dev/nvme2n1p1
mkfs.vfat -F32 /dev/nvme3n1p1
```

---

## Step 9 — Update /etc/fstab and Install Bootloader

### 1. Unmount EFI (if already mounted):

```bash
umount /boot/efi
```

### 2. Get UUID of EFI partition:

```bash
blkid /dev/nvme0n1p1
```

Example output:

```
/dev/nvme0n1p1: UUID="7B1F-36F2" TYPE="vfat"
```

### 3. Update `/etc/fstab`:

```bash
nano /etc/fstab
```

Replace the UUID of the `/boot/efi` entry with the new one:

```
UUID=7B1F-36F2 /boot/efi vfat umask=0077 0 1
```

### 4. Mount and reinstall GRUB:

```bash
mount /boot/efi
dpkg-reconfigure grub-efi-amd64
update-grub
```

Repeat GRUB installation on other EFI partitions if needed.

---

## Step 10 — (Optional) Stop and Clean Up Old RAID1 Array

```bash
mdadm --stop /dev/md0
mdadm --zero-superblock /dev/sda2
mdadm --zero-superblock /dev/sdb2
```

---

## Final Notes

To check RAID status:

```bash
cat /proc/mdstat
```

To verify `/boot/efi` is mounted:

```bash
mount | grep efi
```

Ensure backups are in place before attempting migration on production systems.
