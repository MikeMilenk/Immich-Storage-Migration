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
- **[5. Copy the Data](#5-copy-the-data)**
  - **[5.1 First Copy Pass (Immich Still Running)](#51-first-copy-pass-immich-still-running)**
  - **[5.2 Final Copy Pass (Immich Stopped)](#52-final-copy-pass-immich-stopped)**
- **[6. Switch the Mountpoint](#6-switch-the-mountpoint)**
- **[7. Make the New Mount Permanent](#7-make-the-new-mount-permanent)**
- **[8. Start Immich and Reboot Test](#8-start-immich-and-reboot-test)**
- **[9. Summary](#9-summary)**

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
To find out what to enter next, press `m` to open help.
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

I've highlighted in green the disk we just added, which became `scsi2`. I've also hidden the disk S/N for security reasons. 
![Creating GPT](https://github.com/MikeMilenk/Immich-Storage-Migration/blob/4814a9cf75b8529d2e93cec77b334e0f5a05c4d8/images/3.2-Creating%20GPT.png)

---

# 3. Pass the Physical Disk to the VM

Proxmox's GUI (**Hardware → Add → Hard Disk**) only lets you create a virtual disk on existing Proxmox storage — there's no field for raw passthrough. That requires the CLI, using the disk's stable `by-id` path (not `/dev/sdX`, which can change on reboot):

```bash
qm set [VM_ID] -scsi[N] /dev/disk/by-id/[INTERFACE]-[MODEL]_[SERIAL]
qm config 111
```
![Adding new disk to VM](https://github.com/MikeMilenk/Immich-Storage-Migration/blob/4814a9cf75b8529d2e93cec77b334e0f5a05c4d8/images/3.3-Adding%20disk%20to%20VM%20111.png)

Reboot the VM so it detects the new disk.
![Reboot VM](https://github.com/MikeMilenk/Immich-Storage-Migration/blob/21822aa3bc3877973d61d2807b35c2b748d61f03/images/4-Reboot%20VM.png)

---

# 4. Create the Filesystem and a Temporary Mount

The disk should now appear inside the VM. In the screenshot, I am in the Ubuntu VM running Immich (not the Proxmox host). I ran the `lsblk` command to verify that the new disk was detected (in this case, as `sdc`). I then formatted it as `ext4` and created the `sdc1` partition with the label `immich-data`."

```bash
sudo mkfs.ext4 -L immich-data /dev/sdc1
```
![Create new FS](https://github.com/MikeMilenk/Immich-Storage-Migration/blob/21822aa3bc3877973d61d2807b35c2b748d61f03/images/5.1-Create%20new%20FS.png)

Don't mount straight over the live data — use a temporary mountpoint first, so the new and old storage stay clearly separate during the copy.
In my case, the original mount point is named `immich-storage`, and the new temporary one I'll name `immich-storage-new`

```bash
sudo mkdir /mnt/immich-storage-new
sudo mount /dev/sdc1 /mnt/immich-storage-new
```
![Create new mointpoint](https://github.com/MikeMilenk/Immich-Storage-Migration/blob/21822aa3bc3877973d61d2807b35c2b748d61f03/images/5.2-Create%20new%20dir%20and%20Mountpoint.png)

---

# 5. Copy the Data
  
Copying happens in 2 passes: a long one while Immich stays online, and a short final one after stopping it, so most of the data is already in place and downtime stays minimal.
 
## 5.1 First Copy Pass (Immich Still Running)
  
This pass copies the bulk of the data while Immich keeps running — no downtime yet, but not guaranteed 100% consistent, since Immich may still be writing.
 
```bash
sudo rsync -aHAX --info=progress2 /mnt/immich-storage/ /mnt/immich-storage-new/
```
![Copying data](https://github.com/MikeMilenk/Immich-Storage-Migration/blob/21822aa3bc3877973d61d2807b35c2b748d61f03/images/6-Copying%20data.png)
 
For a large library on a mechanical HDD, this can take hours. In my case, copying almost 800GB of data took 2.5h
 
## 5.2 Final Copy Pass (Immich Stopped)
  
First, stop **Immich** so the source data is frozen. Find the compose file if needed:
 
```bash
sudo find / -iname "docker-compose.yml" 2>/dev/null
```
 
Stop the stack (this only removes the containers, not the data or images):
 
```bash
cd /home/mike/immich-app && sudo docker compose down
sudo docker ps -a   # confirm nothing is left running
```
No containers should appear.

![Finding and stopping Docker](https://github.com/MikeMilenk/Immich-Storage-Migration/blob/21822aa3bc3877973d61d2807b35c2b748d61f03/images/7-finding%20docker%20path%20and%20stopping%20it.png)
 
With the source now frozen, re-sync — adding `--delete` so anything removed from the source is also removed from the copy:
 
```bash
sudo rsync -aHAX --info=progress2 --delete /mnt/immich-storage/ /mnt/immich-storage-new/
```

Since most data is already there, this should be fast. Verify both sides match before moving on:
 
```bash
sudo du -sh /mnt/immich-storage /mnt/immich-storage-new
```
```bash
sudo find /mnt/immich-storage -type f | wc -l
```
```bash
sudo find /mnt/immich-storage-new -type f | wc -l
```

Both the size and file count should be identical.

![Verifying files q-ty](https://github.com/MikeMilenk/Immich-Storage-Migration/blob/21822aa3bc3877973d61d2807b35c2b748d61f03/images/8-verifying%20files%20q-ty.png)

---

# 6. Switch the Mountpoint

```bash
sudo umount /mnt/immich-storage
```
```bash
sudo umount /mnt/immich-storage-new
```
```bash
sudo mount /dev/sdc1 /mnt/immich-storage
```

> If `umount` reports "target is busy," make sure your shell isn't currently sitting inside that directory. In that case, it's better to navigate to your home directory..
```bash
cd ~
```

![Mount new disk into original mountpoint](https://github.com/MikeMilenk/Immich-Storage-Migration/blob/b6403500a2fc470c5db958e30226486acbcaca5e/images/9-mount%20new%20disk%20into%20original%20mount%20point.png)

Now our new `sdc` disk is sitting in the original mount point `/mnt/immich/storage`.

---

# 7. Make the New Mount Permanent

Without this, the new disk won't remount automatically on reboot. Back up first, then swap the old disk's `UUID` for the new one:

```bash
sudo cp /etc/fstab /etc/fstab.bak
```
Enter the fstab editor:
```bash
sudo nano /etc/fstab
```

![Backup fstab](https://github.com/MikeMilenk/Immich-Storage-Migration/blob/21822aa3bc3877973d61d2807b35c2b748d61f03/images/10-backup%20fstab.png)

Edit the line for `/mnt/immich-storage` to use the new filesystem's UUID (from `lsblk -f`)
Save it: `Ctrl+O`, `Enter`
Exit the fstab editor: `Ctrl+X`

Verify that changes were saved:
```bash
cat /etc/fstab
```
Then check it works without rebootingЖ
```bash
sudo mount -a
```

No output from `mount -a` means no errors.

---

# 8. Start Immich and Reboot Test

```bash
cd /home/mike/immich-app && sudo docker compose up -d
```
```bash
sudo docker ps
```

![Start Docker](https://github.com/MikeMilenk/Immich-Storage-Migration/blob/21822aa3bc3877973d61d2807b35c2b748d61f03/images/12-start%20docker.png)

Check the web UI: gallery loads, thumbnails render, a few photos/videos open fine in full size.

Then do one real reboot to confirm the disk mounts and Immich comes back up on its own from a cold start.

```bash
reboot -n
```

In my case, I verified everything through the web interface and the mobile app. Immich is accessible, synchronization is working, and the available storage has been updated from 1.4 TB to 8 TB. Everything is working as expected.

![Immich Mobile App](https://github.com/MikeMilenk/Immich-Storage-Migration/blob/1c25a81342ffca46c9d64c1b8cf9099dd88ceb8e/images/13-Immich%20Mobile%20app.PNG)

Remove the now-unneeded temporary mountpoint. In my case it was `immich-storage-new`:

```bash
sudo rmdir /mnt/immich-storage-new
```

**P.S.** I left the old `ZFS pool` and disk intact and attached to the VM for several days after the migration, keeping them as a safety net rather than cleaning up right away. Some issues — like a subtly corrupted file or a metadata mismatch — might only surface once Immich is actually used in daily practice (browsing albums, search, face recognition, etc.), not just from a quick file count and size comparison. Only once the new disk had proven stable under normal use did I move on to detaching the old disk from the VM and decommissioning the `ZFS pool`.

---

# 9. Summary
- Identified and passed the new physical disk through to the Proxmox VM using its by-id path
- Partitioned GPT on Proxmox, then formatted it as `ext4` from inside the VM
- Mounted it to a temporary path `/mnt/immich-storage-new` and ran a first rsync pass while Immich stayed online
- Stopped Docker, then ran a final rsync pass with `--delete` for a fully consistent copy
- Unmounted both old and new disks, remounted the new disk onto the original `/mnt/immich-storage` path
- Updated `/etc/fstab` with the new disk's `UUID` and verified it with mount `-a`
- Brought Immich back up and confirmed everything worked, including after a full VM reboot
- Left the old ZFS pool and disk intact and attached, as a safety net before eventual cleanup
