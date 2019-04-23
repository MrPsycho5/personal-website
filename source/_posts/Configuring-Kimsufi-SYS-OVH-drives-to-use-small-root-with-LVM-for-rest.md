---
title: LVM for Proxmox on OVH/SoYouStart/Kimsufi
date: 2019-04-23 21:46:52
tags:
  - Proxmox
  - LVM
  - OVH
  - SYS
  - Kimsufi
  - Partitions
  - Drives
categories:
  - cheatsheets
author: Jakub Papuga

---

## Introduction

You may have noticed that OVH's predefined templates are not always what you are looking for. I hate being unable to use all the space for my Proxmox's VMs. This time we will be manually configuring a small RAID protected root partition and LVM for the rest of avaiable space (we are going to backup the VMs anyway).

### My SYS has 3x2TB hard drives. Adjust the article to your needs.

## Reinstalling the server

Start off by ticking the `Custom installation` option.

![SYS Custom Installation](https://i.mrpsycho.pl/selif/2f4e5ja4.png)

Now go ahead and remove the "Remaining Space" partition as we are going to LVM it manually with the rest of the drives.

It is also the time to adjust both the root and swap partition sizes. I'm going with 15GB (15360MB) for my root partition and 8GB (8192MB) for swap. Proxmox 5 fresh installation takes around 7.5GB and Debian around 4GB, so there is plenty of space for the OS.

You can also tick the box on top if for some reason you want to have the root partition only on the first drive. If you leave the box unchecked SYS will create a RAID 1 partition on all drives (3 in my case).

![SYS Partitioning](https://i.mrpsycho.pl/selif/7r10l8g1.png)

On the next page, you can define custom hostname and SSH key if you previously added one.

## Creating partitions

Check the current partition layout by issuing:

```
fdisk -l
```

The output looks like this:

```
Disk /dev/sda: 1.8 TiB, 2000398934016 bytes, 3907029168 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xd4a5d1c5

Device     Boot    Start      End  Sectors Size Id Type
/dev/sda1  *        4096 31459327 31455232  15G fd Linux raid autodetect
/dev/sda2       31459328 48234495 16775168   8G 82 Linux swap / Solaris


Disk /dev/sdb: 1.8 TiB, 2000398934016 bytes, 3907029168 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x7fec845a

Device     Boot    Start      End  Sectors Size Id Type
/dev/sdb1  *        4096 31459327 31455232  15G fd Linux raid autodetect
/dev/sdb2       31459328 48234495 16775168   8G 82 Linux swap / Solaris


Disk /dev/sdc: 1.8 TiB, 2000398934016 bytes, 3907029168 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x8b68cc64

Device     Boot    Start      End  Sectors Size Id Type
/dev/sdc1  *        4096 31459327 31455232  15G fd Linux raid autodetect
/dev/sdc2       31459328 48234495 16775168   8G 82 Linux swap / Solaris


Disk /dev/md1: 15 GiB, 16105013248 bytes, 31455104 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```

It may be a good idea to write down the last sector after which you want to create new partition - `48234495` in my case.

To create new partition on /dev/sda issue:

```
fdisk /dev/sda
```

```
Welcome to fdisk (util-linux 2.29.2).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help):
```

- `n` - to create a new partition
- `p` - to create a primary partition
- `3` - to make the partition third on the drive

Generally the default values are accurate, but SYS creates the first partition on sector 4096, which means sectors 2048-4096 are unused and therefore fdisk will improperly suggest 2048. Add 1 to the sector you previously wrote down (`48234495`+`1`=`48234496` in my case). As for the last sector, fdisk should be correct.

```
Command (m for help): n
Partition type
   p   primary (2 primary, 0 extended, 2 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (3,4, default 3):
First sector (2048-3907029167, default 2048): 48234496
Last sector, +sectors or +size{K,M,G,T,P} (48234496-3907029167, default 3907029167):

Created a new partition 3 of type 'Linux' and of size 1.8 TiB.
```

### If you have created a 1MiB partition read this paragraph again.

To prepare the partition to be used by LVM use the following two commands.

- `t` - to change the partition type
- `8e` - to change the partition type to LVM

```
Command (m for help): t
Partition number (1-3, default 3): 3
Partition type (type L to list all types): 8e

Changed type of partition 'Linux' to 'Linux LVM'.
```

You can now:

- `w` - to write changes to the disk and quit
- `q` - to quit without writing the changes

Before you reboot the node redo the steps for `/dev/sdb`, `/dev/sdc` and any other drives you may have.

```
reboot
```

## Create Physical Volume

Use `pvcreate` on each partition to create Physical Volumes
```
pvcreate /dev/sda3
pvcreate /dev/sdb3
pvcreate /dev/sdc3
```

If you're asking yourself why we didn't format the partitions or create a filesystem, don't worry. That step comes later.

## Create Volume Group

Now that we have Physical Volumes defined we can create a Volume Group. `vgpool` can be any name you want, however it is recommended to put `vg` at the front, so in the future you will know it is a volume group. You can create the Volume Group will all the Physical Volumes at once.

```
vgcreate vgpool /dev/sda3 /dev/sdb3 /dev/sdc3
```

## Create Logical Volume

To create the Logical Volume with all the available space issue:

```
lvcreate -l 100%FREE -n lvpve  vgpool
```

Again, `lvpve` can be anything you want.

If you want to create a smaller Logical Volume you can use `vgdisplay` to look up the available space. For example

- `lvcreate -L 30G -n lvpve vgpool` will create a 30GB `lvpve` Logical Volume from `vgpool`.
- `lvextend -L80G /dev/vgpool/lvpve` will excent the partition <u>to</u> 80GB
- `lvextend -L+80G /dev/vgpool/lvpve` will excent the partition <u>by</u> 8GB

## Creating Filesystem

Simply:

```
mkfs.ext4 /dev/vgpool/lvpve
```

It may take a few seconds to execute. Be patient.

```
mke2fs 1.43.4 (31-Jan-2017)
Creating filesystem with 1447047168 4k blocks and 180883456 inodes
Filesystem UUID: 2c62a995-f45f-46e9-a245-597bc97c4979
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
        4096000, 7962624, 11239424, 20480000, 23887872, 71663616, 78675968,
        102400000, 214990848, 512000000, 550731776, 644972544

Allocating group tables: done
Writing inode tables: done
Creating journal (262144 blocks): done
Writing superblocks and filesystem accounting information: done
```

In the future, if you decide to extend the Logical Volume you will also need to resize the filesystem: `resize2fs /dev/vgpool/lvpve`

## Mounting the filesystem

Simply make a directory where you want the LV to be mounted and mount it there.

```
mkdir /mnt/pve
mount -t ext4 /dev/vgpool/lvpve /mnt/pve
```

...as for Proxmox goes. You may want to go to `Datacenter -> Storage -> Add -> Directory` to add the freshly mounted filesystem.

![Proxmox Add Directory](https://i.mrpsycho.pl/selif/0gonqlt3.png)

### Make sure to disable the `local` storage in Proxmox!
