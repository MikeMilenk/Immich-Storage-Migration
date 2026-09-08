# Immich-Storage-Migration
This guide shows how to migrate an Immich library off a ZFS pool onto a single physical HDD passed through directly to a Proxmox VM

Background: in an earlier [guide](https://github.com/MikeMilenk/Immich-deployment.git), I built a ZFS pool (immich-zfs) out of 2 disks to back the Immich VM's storage. This guide undoes that — all data moves off the ZFS pool onto a single new physical disk, removing the ZFS layer entirely in favor of a plain disk.

WARNING: This process involves formatting a disk and editing /etc/fstab. Always verify the exact disk by model, serial number, and size before running any destructive command. Do not proceed on a production system without a way to identify your disks by /dev/disk/by-id/.
