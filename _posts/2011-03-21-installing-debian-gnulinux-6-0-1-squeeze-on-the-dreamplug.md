---
title: 'Installing Debian GNU/Linux 6.0.1 (squeeze) on the DreamPlug.'
date: '2011-03-21T22:03:10+00:00'
author: deaves
layout: post
permalink: /2011/03/installing-debian-gnulinux-6-0-1-squeeze-on-the-dreamplug/
categories:
    - 'Linux Admin'
tags:
    - debian
    - dreamplug
    - installing
    - linux
---

After about a month of waiting the [ DreamPlug and JTAG module](http://www.globalscaletechnologies.com/p-41-dreamplug-devkit.aspx) that I ordered was finally delivered. So far as a dedicated home server goes, the specs, physical size, price and longterm energy saving made it almost a no-brainer to purchase. However once it arrived it did not take me very long to realize that the preinstalled Ubuntu version on it was already obsolete and was not going to be supported. Without fully realizing what I was doing I quickly bricked the HostOS in an attempt to force a release update via a “do-release-upgrade” courtesy of the update-manager-core utilities. So there I was, frustrated, with a new brick of a system, and was forced to try to do a restore only to find out that there was no documentation out on the Internet to assist. I decided to document what all it took to get the DreamPlug back up and working and to install Debian on it, so hopefully I can save someone from all the pain and anguish I suffered while trying to get past the learning curve it took to get the system up and working..

## Before we begin

There are a few websites and concepts that are absolutely critical with getting the DreamPlug working…

- <http://code.google.com/p/dreamplug/>  
    *This site contains everything you need to create a USB recovery stick to restore the host OS on the device. After installing Debian I needed to boot from the USB recovery stick to modify the boot partition and make changes to /etc/fstab. !Make sure you download everything on this site and keep it in a safe place!*
- <http://www.cyrius.com/debian/kirkwood/sheevaplug/install.html>  
    *This site has very good information on working on the guru/shivaplug as well as working with U-Boot. Although a lot of the information on this site is superfluous and obsolete, it still contains a lot of awesome information which was absolutely critical for figuring things out.*
- [ ftp://ftp.debian.org/debian/dists/squeeze/main/installer-armel/current/images/kirkwood/netboot/marvell/sheevaplug/uInitrd](ftp://ftp.debian.org/debian/dists/squeeze/main/installer-armel/current/images/kirkwood/netboot/marvell/sheevaplug/uInitrd)  
    *uInitrd is the Debian installer which you will need if you want to get Debian working on the DreamPlug. !Do not download “uImage” file from there; because it does not work and you will be using the the uImage file from the code.google.com/p/dreamplug/ site!*
- <http://git.marvell.com/>  
    *Marvell’s source tree. If you wish to recompile your kernel I would recommend using the kernel from here /<sup><span style="text-decoration: underline;">git clone git://git.marvell.com/orion.git</span></sup>/ instead of kernel.org (at least untill the DreamPlug becomes better supported).*

One thing that tripped me up for a while was understanding how everything works together on the DreamPlug. These are things that are are universal between the DreamPlug the ShivaPlug, GuruPlug and several other ARM systems. U-Boot – U-Boot is nothing more then a boot loader; in the same way Grub or Lilo is. Its the closest you will be getting to a BIOS and being comfortable with it is critical in case something goes wrong and you need to do some troubleshooting.

U-Boot keeps its own set of environmental variables that tell it what and how to do things. To view, modify and delete those variables you need to use: printenv, setenv, saveenv … Before making any changes, I strongly advise doing a printenv and making a backup of what is listed.

The internal storage device is tied to USB, to access things like the built in storage, SD cards or memory sticks from U-Boot you need to start USB by doing a: usb start … To view connected usb drives and partitions do a: “*usb devices*” and “*usb partitions*” . This is important when trying to figure out how to reference devices to load your kernel.

Researching on the Internet I found some versions of U-Boot support kernels located on EXT2 partitions. The version of U-Boot that comes with the DreamPlug <sup>U-Boot 2011.06-02334-g8f495d9-dirty (Mar 01 2011 – 06:57:05)</sup> does not support EXT2, only FAT, so your /boot partition will need to be setup and formated accordingly. This will make the Debian install a little tricky because the Debian installer does not like FAT partitions; which I will explain how I got around later.

## Creating and booting from a recovery USB stick

As mentioned earlier, you will want to download everything from <http://code.google.com/p/dreamplug/> because its got everything you need to create a recovery USB stick. Before extracting though you need to run fdisk against it and create 2 partitions: a FAT partition for uImage and uInitrd and another for the rootfs. The following is how I setup my partitions:

```
Mount Point    Device    Boot   Size   Id  System
/boot          /dev/sde1   *    165M    b  W95 FAT32
/              /dev/sde2        3.8G   83  Linux
```

Note: *I would recommend making the FAT partition at least 100M because you will also want to store an archive of /lib/modules/`uname -r` and /lib/firmware from the recovery USB on there later.*

Next, format the FAT32 partition as VFAT and copy **uImage** (from the google code site) and **uInitrc** from (from Debian’s site) to it. Format the Linux partition as EXT4 and extract **dreamplug\_rootfs.zip** to it; making sure the directory tree starts at / … Go ahead and do a sync and umount on your USB stick and insert in into the DreamPlug.

The following is how to tell U-Boot how to boot into Ubuntu off the recovery stick you just created:

```
Marvell>> usb start
Marvell>> fatload usb 1:1 0x00800000 /uImage
Marvell>> setenv bootargs_console console=ttyS0,115200
Marvell>> setenv bootargs_root root=/dev/sdc2 rootdelay=10
Marvell>> setenv bootcmd 'setenv bootargs ${bootargs_console} ${bootargs_root}; run bootcmd_usb; bootm 0x00800000'
Marvell>> run bootcmd
```

*Sometimes the USB reference on the fatload statement and the Linux root partition will differ; use the “usb partitions” command to confirm the device id that U-Boot uses to recognize the recovery stick.*

Once you’ve booted in to Ubuntu, login as **root**/**nosoup4u** and setup your partitions as follows:

```
Disk /dev/sda: 1977 MB, 1977614336 bytes
61 heads, 62 sectors/track, 1021 cylinders
Units = cylinders of 3782 * 512 = 1936384 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x0004e16e

Device    Boot      Start         End      Blocks   Id  System
/dev/sda1   *           1          85      160704    b  W95 FAT32
/dev/sda2              86         946     1628151   83  Linux
/dev/sda4             947        1021      141825   82  Linux swap / Solaris
```

Go ahead and format the partitions as VFAT and EXT4. If you want to restore Ubuntu just copy uImage to /dev/sda1, restore dreamplug\_rootfs to /dev/sda2 and make sure the bootcmd variable is setup properly in U-Boot (*Ref: Finalizing U-Boot*). If you are going to install Debian go ahead and copy uImage and iInitrd to /dev/sda1; you can skip doing any restores on /dev/sda2. You will also need to do a “*tar cfjv modules-`uname -r`.tar.bz2 /lib/modules/`uname -r`*” and “*tar cfjv `hostname`-firmware.tar.bz2 /lib/firmware*” copy the files to /dev/sda1; as we will be restoring it after Debian is installed.

## Installing Debian on the DreamPlug

Once our partitions are preped boot back into U-Boot and run the following to start the Debian install:

```
Marvell>> usb start
Marvell>> fatload usb 0:1 0x00800000 /uImage
Marvell>> fatload usb 0:1 0x01100000 /uInitrd
Marvell>> setenv bootargs console=ttyS0,115200n8 base-installer/initramfs-tools/driver-policy=most
Marvell>> bootm 0x00800000 0x01100000
```

To reiterate from earlier, you are booting the kernel (*uImage*) posted at code.google.com/p/dreamplug and are using the Debian armel install image (*uInitrd*), after the setup you will continue to boot using the same kernel uImage. Follow all the steps as usual during the install with one exception; when you are prompted to setup your disk partitions select manual setup. Make sure you leave sda1 alone, sda2 mounts to / and sda4 is swap. Continue and let the setup complete as normal and then drop to a shell when the install completes. As a finalizing touch you need to do 2 things: modify /etc/fstab on sda2 and restore the two archives you made earlier of *modules-\*.tar.bz2* and *`hostname`-firmware.tar.bz2* on sda2.

## Finalizing U-Boot

Once you have Debian installed make sure you have modified your /etc/fstab you also still need to tell U-Boot what and how to boot the system. To boot there are 2 critical variables that are required: ***bootargs*** (which is passed to the kernel) and ***bootcmd*** (which is automatically ran when the system is powered on). The following is how I set my U-Boot vars:

```
Marvell>> setenv bootcmd_usb 'usb start; fatload usb 0:1 0x00800000 /uImage'
Marvell>> setenv bootargs_console console=ttyS0,115200
Marvell>> setenv bootargs_root root=/dev/sda2 rootdelay=10
Marvell>> setenv bootcmd 'setenv bootargs ${bootargs_console} ${bootargs_root}; run bootcmd_usb; bootm 0x00800000'
Marvell>> saveenv
```

Feel free to customize and clean up your env variables, if you need to delete a variable just do a “*setenv &lt;variable&gt;*“. Also nothing you change will be committed unless you do a “*saveenv*“. It never hurts to experiment a little, the quicker you lean how U-Boot works the easier things will be to fix if something breaks.

## Getting things working

To get wireless working you will need to compile the uaptuil source code and copy the binary to /usr/local/sbin. You can download the source code from the following message board posting: [ http://plugcomputer.org/plugforum/index.php?topic=2196.msg13114#msg13114](http://plugcomputer.org/plugforum/index.php?topic=2196.msg13114#msg13114).

Additionally, for your reference, the following is a copy of my fstab:

```
# /etc/fstab: static file system information.
#
# <file system>  <mount point>   <type>  <options>      <dump>  <pass>

proc       /proc    proc    defaults                                0       0
/dev/sda2  /        ext4    errors=remount-ro                       0       1
/dev/sda1  /boot    vfat    auto,owner,rw,uid=0,gid=0,noexec        0       0
tmpfs      /tmp     tmpfs   defaults,noatime,nodiratime,mode=1777   0       0
/dev/sda4  none     swap    sw                                      0       0
```
