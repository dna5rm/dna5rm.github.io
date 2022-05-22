---
title: 'LUKS Quick Examples'
date: '2014-08-18T13:20:24+00:00'
author: deaves
layout: post
permalink: /2014/08/luks-quick-examples/
categories:
    - 'Linux Admin'
    - 'Linux Security'
tags:
    - examples
    - linux
    - luks
---

## Required Debian/Ubuntu Packages

- **dmsetup** *Linux Kernel Device Mapper userspace library*
- **cryptsetup-bin** *Disk encryption support – command line tools*
- **tcplay** *Free and simple TrueCrypt Implementation based on dm-crypt*

## Filesystem Encryption

- **cryptsetup** -cipher aes-xts-plain64 -key-size 512 -verify-passphrase *luksFormat* /dev/sdb1

### The LUKS-formatting command above has the following options:
- -verify-passphrase – ensures the passphrase is entered twice to avoid an incorrect passphrase being used
- -c aes -s 256 – uses 256-bit AES encryption
- -h sha256 – uses the 256-bit SHA hashing algorithm

## Creating a Filesystem

- **cryptsetup** *luksOpen* /dev/sdb1 16GB
- **mkfs** -t *ext3* -m 1 -O dir_index,filetype,sparse_super /dev/mapper/*16GB*

### The mkfs options above are as follows:

- -t ext3 – create an ext3 filesystem
- -m 1 – reduce the reserved super-user space down from the default of 5% to 1% of the total size – useful for large filesystems
- -O dir\_index – speed-up lookups in large directories
- -O filetype – store filetype info in directories
- -O sparse\_super – create fewer superblock backup copies – useful for large filesystems

## Mounting a Filesystem

- **cryptsetup** *luksOpen* /dev/sdb1 16GB
- **mount** /dev/mapper/*16GB* /mnt

### To mount a truecrypt partition:

- **tcplay** -m *16GB* -d */dev/sdc1*
- **dmsetup** remove *16GB*

## Change Passwords on a Filesystem

LUKS supports eight key slots per partition.

### To add and remove keys from the slots:

- **cryptsetup** *luksAddKey*
/and/
- **cryptsetup** *luksRemoveKey*

### Which slots have keys:

- **cryptsetup** *luksDump*

## Headers on a Filesystem

- **cryptsetup** *luksHeaderBackup* /dev/sdb1 -header-backup-file /tmp/somefile

Replace *luksHeaderBackup* with *luksHeaderRestore* to restore the old keys again.
* Note that the header backup should be saved to a secure place (preferably another LUKS partition on a USB stick)

## Unmount a Filesystem

- Use **umount** first then,</dir>**cryptsetup** *luksClose* 16GB
/or/
- **dmsetup** *remove* 16GB
/or/
- **dmsetup** *remove\_all*\* dmsetup remove\_all will flush all mapped block devices.

## Source & Additional Documentation

- https://help.ubuntu.com/community/EncryptedFilesystemsOnRemovableStorage
- http://superuser.com/questions/431820/how-to-change-pass-phrase-of-full-disk-encryption
- http://askubuntu.com/questions/95137/how-to-change-luks-passphrase
- http://www.linuxcommand.org/man\_pages/cryptsetup8.html
