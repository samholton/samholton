---
layout: post
title: "Backup USB Jump Drive to Disk Image"
date: 2023-09-07 17:31
categories: Linux
tags: [backup]
---

I recently had the need to create a backup image of a portable USB device to store for safe keeping. This was not a bootable drive, just a drive with files on it. Yeah, I could have copied the files from the drive, or I could have create a zip archive from them. But I wanted to create an image as I wasn't really sure of the contents of the drive (it was an update stick for a home arcade machine - future post).

## Create the Backup

1. Find the source device with `lsblk` (you could check dmesg or disk utility or any other method)

    ```console
    root@desktop:/home/sam# lsblk
    NAME                   MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
    sda                      8:0    0 232.9G  0 disk  
    ├─sda1                   8:1    0   243M  0 part  /boot
    ├─sda2                   8:2    0     1K  0 part  
    └─sda5                   8:5    0 232.6G  0 part  
    └─sdb5_crypt         253:0    0 232.6G  0 crypt 
        └─desktop--vg-root 253:1    0 232.6G  0 lvm   /
    sdb                      8:16   1  28.8G  0 disk  
    └─sdb1                   8:17   1  28.8G  0 part  /media/sam/A38A-3083
    ```

1. Create the backup using `dd`

    ```console
    dd if=/dev/sdb of=/home/sam/gt-go.img bs=4M status=progress
    ```

1. Store the image for safe keeping


## Mount the Image

The following describes how to mount the image (read only in this case) and retrieve files from it.

1. Find the block size and offset

    ```console
    fdisk -l /home/sam/gt-go.img 
    Disk /home/sam/gt-go.img: 28.82 GiB, 30943995904 bytes, 60437492 sectors
    Units: sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disklabel type: dos
    Disk identifier: 0x647fdcd9

    Device               Boot Start      End  Sectors  Size Id Type
    /home/sam/gt-go.img1       2048 60437491 60435444 28.8G  b W95 FAT32
    ```

    In this case the block size is `512` and the offset is `2048`

    Could also use the `file` command to find the start sector

    ```console
    file /home/sam/gt-go.img 
    /home/sam/gt-go.img: DOS/MBR boot sector; partition 1 : ID=0xb, start-CHS (0x1,0,1), end-CHS (0x346,31,20), startsector 2048, 60435444 sectors, extended partition table (last)
    ```

1. Calculate the offset using block size and start sector from above

    ```console
    512 * 2048 = 1048576
    ```

1. Mount the image

    ```console
    mkdir /mnt/iso
    mount -o ro,loop,offset=1048576 /home/sam/gt-go.img /mnt/iso

    ls -l /mnt/iso
    ```
