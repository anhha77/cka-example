# NFS Server config

This guide assume you have already added disk to your server or VM

## Identify target disk

```
lsblk
```

Output of `lsblk`:

```text
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda       8:0    0  160G  0 disk
├─sda1    8:1    0  159G  0 part /
├─sda14   8:14   0    4M  0 part
├─sda15   8:15   0  106M  0 part /boot/efi
└─sda16   8:16   0  913M  0 part /boot
sdb       8:16   0   20G  0 disk
```

### Summary

| Device  | Size | Type | Mountpoint    |
| ------- | ---- | ---- | ------------- |
| `sda`   | 160G | disk | —             |
| `sda1`  | 159G | part | `/`           |
| `sda14` | 4M   | part | —             |
| `sda15` | 106M | part | `/boot/efi`   |
| `sda16` | 913M | part | `/boot`       |
| `sdb`   | 20G  | disk | — (unmounted) |

**Note:** `sdb` (20G) is a raw disk with no partitions or mountpoint — available for use (e.g., new partition, PV, or filesystem).

## Create partition using fdisk

```
sudo fdisk /dev/sdb
```

It will output

```text
Welcome to fdisk (util-linux 2.40.2).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Command (m for help):
```

```text
Command (m for help): m

Help:

  GPT
   M   enter protective/hybrid MBR

  Generic
   d   delete a partition
   F   list free unpartitioned space
   l   list known partition types
   n   add a new partition
   p   print the partition table
   t   change a partition type
   v   verify the partition table
   i   print information about a partition

  Misc
   m   print this menu
   x   extra functionality (experts only)

  Script
   I   load disk layout from sfdisk script file
   O   dump disk layout to sfdisk script file

  Save & Exit
   w   write table to disk and exit
   q   quit without saving changes

  Create a new label
   g   create a new empty GPT partition table
   G   create a new empty SGI (IRIX) partition table
   o   create a new empty MBR (DOS) partition table
   s   create a new empty Sun partition table

Command (m for help):
```

We need to enter g->n then choose default for 3 sector then hit w

Check partition

```
lsblk
```

## Format disk

```
mkfs -t ext4 /dev/sdb1
```

## Mount disk to folder in nfs server

```
sudo nano /etc/fstab
```

```text
# <file system>          <mount point>     <type>  <options>                            <dump>  <pass>
LABEL=cloudimg-rootfs    /                 ext4    discard,commit=30,errors=remount-ro  0       1
LABEL=BOOT               /boot             ext4    defaults                             0       2
LABEL=UEFI               /boot/efi         vfat    umask=0077                           0       1
/dev/sdb1                /mnt/nfs-server   ext4    defaults                             0       0
```

### Entries

| File System             | Mount Point       | Type | Options                               | Dump | Pass |
| ----------------------- | ----------------- | ---- | ------------------------------------- | ---- | ---- |
| `LABEL=cloudimg-rootfs` | `/`               | ext4 | `discard,commit=30,errors=remount-ro` | 0    | 1    |
| `LABEL=BOOT`            | `/boot`           | ext4 | `defaults`                            | 0    | 2    |
| `LABEL=UEFI`            | `/boot/efi`       | vfat | `umask=0077`                          | 0    | 1    |
| `/dev/sdb1`             | `/mnt/nfs-server` | ext4 | `defaults`                            | 0    | 0    |

```
systemctl deamon-reload
mount -a
```

After mount

```
lsblk
```

Return result:

```text
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
fd0       2:0    1    4K  0 disk
sda       8:0    0   50G  0 disk
├─sda1    8:1    0   49G  0 part /
├─sda14   8:14   0    4M  0 part
├─sda15   8:15   0  106M  0 part /boot/efi
└─sda16 259:0    0  913M  0 part /boot
sdb       8:16   0  200G  0 disk
└─sdb1    8:17   0  200G  0 part /mnt/nfs-server
sr0      11:0    1 1024M  0 rom
```

## Config nfs server and nfs client

### NFS Server

```
sudo apt update
sudo apt -y install nfs-kernel-server
```

Then config the /etc/exports file in nfs server

```
sudo nano /etc/fstab
```

The access control list for filesystems exported to NFS clients. See `exports(5)`.

```text
# /etc/exports: the access control list for filesystems which may be exported
#               to NFS clients.  See exports(5).
#
# Example for NFSv2 and NFSv3:
# /srv/homes       hostname1(rw,sync,no_subtree_check) hostname2(ro,sync,no_subtree_check)
#
# Example for NFSv4:
# /srv/nfs4        gss/krb5i(rw,sync,fsid=0,crossmnt,no_subtree_check)
# /srv/nfs4/homes  gss/krb5i(rw,sync,no_subtree_check)
#
/mnt/nfs-server  10.0.0.164/24(rw,no_root_squash,sync)  10.0.0.108/24(rw,no_root_squash,sync)  10.0.0.132/24(rw,no_root_squash,sync)
```

### Export

**Path:** `/mnt/nfs-server`

| Client          | Options                  |
| --------------- | ------------------------ |
| `10.0.0.164/24` | `rw,no_root_squash,sync` |
| `10.0.0.108/24` | `rw,no_root_squash,sync` |
| `10.0.0.132/24` | `rw,no_root_squash,sync` |

### Option meanings

- **rw** — read/write access
- **no_root_squash** — remote root is treated as local root (no UID remap)
- **sync** — writes committed to stable storage before the reply
  > **Note on `/24`:** With CIDR notation, `10.0.0.164/24` grants access to the entire `10.0.0.0/24` subnet, not just that single host — the host bits are masked off. All three entries therefore resolve to the same subnet. If you intend to restrict to specific hosts, use `/32` (e.g. `10.0.0.164/32`) or a bare IP `10.0.0.164`.

After editing, apply with:

```bash
sudo exportfs -ra        # re-export all
sudo exportfs -v         # verify active exports
```

## NFS Client

Config in every node you want to mount with nfs storage

```
sudo apt update
sudo apt -y install nfs-common
```
