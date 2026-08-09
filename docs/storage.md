# Storage Configuration

## Seagate 750GB HDD — Bulk Storage

Added a Seagate 698GB HDD to the Proxmox host for VM/container storage and files.

### Steps

1. **Identified the drive** (confirmed device before any destructive action):
   ```bash
   lsblk -o NAME,SIZE,MODEL,MOUNTPOINTS
   ```
   Confirmed `sda` (698.6G) was the blank Seagate — *not* the `nvme0n1` Proxmox
   system drive. Always confirm the device before formatting.

2. **Partitioned** the disk:
   ```bash
   fdisk /dev/sda
   # n (new partition), accept defaults for full-disk partition, w (write)
   ```

3. **Formatted** with ext4:
   ```bash
   mkfs.ext4 /dev/sda1
   ```

4. **Got the filesystem UUID:**
   ```bash
   blkid /dev/sda1
   # UUID="2ff1834c-..." TYPE="ext4"
   ```
   Note: use the filesystem `UUID`, not the `PARTUUID`.

5. **Created a mount point and added a persistent mount** in `/etc/fstab`:
   ```bash
   mkdir -p /mnt/seagate
   ```
   Added to `/etc/fstab` (no quotes around the UUID):
   ```
   UUID=2ff1834c-... /mnt/seagate ext4 defaults 0 2
   ```

6. **Mounted and verified:**
   ```bash
   systemctl daemon-reload
   mount -a
   df -h /mnt/seagate
   ```
   Result: `/dev/sda1  687G  /mnt/seagate` — mounted and auto-mounts on boot.

7. **Added as Proxmox storage** (Datacenter -> Storage -> Add -> Directory,
   path `/mnt/seagate`), or via CLI:
   ```bash
   pvesm add dir seagate --path /mnt/seagate --content images,rootdir,iso,vztmpl,backup
   ```

### Lessons

- Creating a partition does **not** format it — `mkfs` is a separate, required step.
- fstab uses the filesystem `UUID` (from `blkid`), with **no quotes** and **no angle brackets**.
- A single spinning HDD has no redundancy — fine as a stopgap, but important data
  needs backups (single drive = single point of failure).
