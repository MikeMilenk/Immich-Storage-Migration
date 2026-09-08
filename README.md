# Immich Storage Migration: ZFS Zvol → Physical HDD (ext4)

This guide shows how to migrate an [Immich](https://immich.app/) library off a ZFS pool onto a single physical HDD passed through directly to a Proxmox VM, using plain ext4.

**Background:** in an earlier [guide](https://github.com/MikeMilenk/Immich-deployment.git), I built a ZFS pool `immich-zfs` out of 2 disks to back the Immich VM's storage. This guide undoes that — all data moves off the ZFS pool onto a single new physical disk, removing the ZFS layer entirely in favor of a plain disk. Here you can see my ZFS pool, which combines `sdd` and `sdc`."
![ZFS Pool](https://github.com/MikeMilenk/Immich-Storage-Migration/blob/63ff337336aeba9e16e1d58daf8a71052c057fc7/images/1-ZFS%20Pool.png)


**Setup:**
- Host: Proxmox VE
- VM: Ubuntu Linux, running Immich in Docker
- Old storage: ZFS pool (2 HDDs)
- New storage: a single 8TB physical HDD

## WARNING: This process involves formatting a disk and editing `/etc/fstab`. Always verify the exact disk by model, serial number, and size before running any destructive command. Do not proceed on a production system without a way to identify your disks by `/dev/disk/by-id/`.

---

## Table of Contents:

- **[1. Identify the Disks](#1-identify-the-disks)**
- **[2. Partition the New Disk on Proxmox](#2-partition-the-new-disk-on-proxmox)**
- **[3. Pass the Physical Disk to the VM](#3-pass-the-physical-disk-to-the-vm)**
- **[4. Create the Filesystem and a Temporary Mount](#4-create-the-filesystem-and-a-temporary-mount)**
- **[5. First Copy Pass (Immich Still Running)](#5-first-copy-pass-immich-still-running)**
- **[6. Stop Immich](#6-stop-immich)**
- **[7. Final Copy Pass (Immich Stopped)](#7-final-copy-pass-immich-stopped)**
- **[8. Switch the Mountpoint](#8-switch-the-mountpoint)**
- **[9. Update /etc/fstab](#9-update-etcfstab)**
- **[10. Start Immich and Reboot Test](#10-start-immich-and-reboot-test)**
- **[11. Summary](#11-summary)**

---

# 1. Identify the Disks

Before touching anything, check what's currently there — the ZFS pool disks and the new target disk — by size, model, and serial, not just by `/dev/sdX` name.

```bash
lsblk -o NAME,SIZE,MODEL,SERIAL,FSTYPE,MOUNTPOINTS
```
![ZFS Pool](https://github.com/MikeMilenk/Immich-Storage-Migration/blob/63ff337336aeba9e16e1d58daf8a71052c057fc7/images/1-ZFS%20Pool.png)

---

# 2. Partition the New Disk on Proxmox

**This only creates a partition table — no filesystem yet.**

Clear old signatures and lay down a fresh GPT table:

```bash
wipefs -a /dev/sde
fdisk /dev/sde
```
To find out what to enter next, press 'm' to open help.
![Wipe disk and enter fdisk](https://github.com/MikeMilenk/Immich-Storage-Migration/blob/4814a9cf75b8529d2e93cec77b334e0f5a05c4d8/images/3.1-Wipe%20and%20Enter%20fdisk.png)

Inside `fdisk`:

```
g       # new GPT table
n       # new partition
Enter   # accept default partition number
Enter   # accept default first sector
Enter   # accept default last sector (whole disk)
w       # write changes
```

I've highlighted in green the disk we just added, which became **scsi2**. I've also hidden the disk S/N for security reasons. 
![Creating GPT](https://github.com/MikeMilenk/Immich-Storage-Migration/blob/4814a9cf75b8529d2e93cec77b334e0f5a05c4d8/images/3.2-Creating%20GPT.png)

---

# 3. Pass the Physical Disk to the VM

Proxmox's GUI (**Hardware → Add → Hard Disk**) only lets you create a virtual disk on existing Proxmox storage — there's no field for raw passthrough. That requires the CLI, using the disk's stable `by-id` path (not `/dev/sdX`, which can change on reboot):

```bash
qm set [VM_ID] -scsi[N] /dev/disk/by-id/[INTERFACE]-[MODEL]_[SERIAL]
qm config 111
```
![Adding new disk to VM](https://github.com/MikeMilenk/Immich-Storage-Migration/blob/4814a9cf75b8529d2e93cec77b334e0f5a05c4d8/images/3.3-Adding%20disk%20to%20VM%20111.png)


Reboot the VM so it detects the new disk:

```bash
reboot -n
```

The disk should now appear inside the VM (in this case, as `sdc`).

---

# 4. Create the Filesystem and a Temporary Mount

[#4-create-the-filesystem-and-a-temporary-mount](#4-create-the-filesystem-and-a-temporary-mount)

Inside the VM:

```bash
sudo mkfs.ext4 -L immich-data /dev/sdc1
```

Don't mount straight over the live data — use a temporary mountpoint first, so the new and old storage stay clearly separate during the copy:

```bash
sudo mkdir /mnt/immich-storage-new
sudo mount /dev/sdc1 /mnt/immich-storage-new
```

---

# 5. First Copy Pass (Immich Still Running)

[#5-first-copy-pass-immich-still-running](#5-first-copy-pass-immich-still-running)

This pass copies the bulk of the data while Immich keeps running — no downtime yet, but not guaranteed 100% consistent, since Immich may still be writing.

```bash
sudo rsync -aHAX --info=progress2 /mnt/immich-storage/ /mnt/immich-storage-new/
```

For a large library on a mechanical HDD, this can take hours.

---

# 6. Stop Immich

[#6-stop-immich](#6-stop-immich)

Find the compose file if needed:

```bash
sudo find / -iname "docker-compose.yml" 2>/dev/null
```

Stop the stack (this only removes the containers, not the data or images):

```bash
cd /home/mike/immich-app && sudo docker compose down
sudo docker ps -a   # confirm nothing is left running
```

---

# 7. Final Copy Pass (Immich Stopped)

[#7-final-copy-pass-immich-stopped](#7-final-copy-pass-immich-stopped)

With the source now frozen, re-sync — adding `--delete` so anything removed from the source is also removed from the copy:

```bash
sudo rsync -aHAX --info=progress2 --delete /mnt/immich-storage/ /mnt/immich-storage-new/
```

Since most data is already there, this should be fast. Verify both sides match before moving on:

```bash
sudo du -sh /mnt/immich-storage /mnt/immich-storage-new
sudo find /mnt/immich-storage -type f | wc -l
sudo find /mnt/immich-storage-new -type f | wc -l
```

Both the size and file count should be identical.

---

# 8. Switch the Mountpoint

[#8-switch-the-mountpoint](#8-switch-the-mountpoint)

```bash
cd ~
sudo umount /mnt/immich-storage
sudo umount /mnt/immich-storage-new
sudo mount /dev/sdc1 /mnt/immich-storage
```

> If `umount` reports "target is busy," make sure your shell isn't currently sitting inside that directory.

---

# 9. Update /etc/fstab

[#9-update-etcfstab](#9-update-etcfstab)

Without this, the new disk won't remount automatically on reboot. Back up first, then swap the old disk's UUID for the new one:

```bash
sudo cp /etc/fstab /etc/fstab.bak
sudo nano /etc/fstab
```

Edit the line for `/mnt/immich-storage` to use the new filesystem's UUID (from `lsblk -f`), save (`Ctrl+O`, `Enter`), exit (`Ctrl+X`).

Verify, then test the syntax without rebooting:

```bash
cat /etc/fstab
sudo mount -a
```

No output from `mount -a` means no errors.

---

# 10. Start Immich and Reboot Test

[#10-start-immich-and-reboot-test](#10-start-immich-and-reboot-test)

```bash
cd /home/mike/immich-app && sudo docker compose up -d
sudo docker ps
```

Check the web UI: gallery loads, thumbnails render, a few photos/videos open fine in full size.

Then do one real reboot to confirm the disk mounts and Immich comes back up on its own from a cold start — this is a better test than `mount -a` alone:

```bash
reboot -n
```

---

# 11. Summary

[#11-summary](#11-summary)

The Immich library was migrated off the two-disk ZFS pool (pool → zvol → VM passthrough) onto a single physical HDD, passed through to the VM directly and formatted as plain ext4. The disk was partitioned and passed through from Proxmox, then formatted and mounted from inside the VM. Data was copied over in two rsync passes — a long first pass while Immich stayed online, followed by a short final pass after stopping Docker to guarantee a consistent copy. The mountpoint was then switched over, `/etc/fstab` updated to the new disk's UUID, and Immich brought back up and confirmed working, including after a full VM reboot.

The old ZFS-backed disk was left attached and untouched at this point, kept as a safety net until the new disk has proven stable under normal, everyday use. Detaching it from the VM and decommissioning the ZFS pool is intentionally left as a separate, later step — not something to rush right after the migration.
# Immich-Storage-Migration
This guide shows how to migrate an Immich library off a ZFS pool onto a single physical HDD passed through directly to a Proxmox VM

Background: in an earlier [guide](https://github.com/MikeMilenk/Immich-deployment.git), I built a ZFS pool (immich-zfs) out of 2 disks to back the Immich VM's storage. This guide undoes that — all data moves off the ZFS pool onto a single new physical disk, removing the ZFS layer entirely in favor of a plain disk.

WARNING: This process involves formatting a disk and editing /etc/fstab. Always verify the exact disk by model, serial number, and size before running any destructive command. Do not proceed on a production system without a way to identify your disks by /dev/disk/by-id/.
