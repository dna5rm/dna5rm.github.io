---
title: 'Configuring netatalk under Ubuntu.'
date: '2010-11-27T10:24:04+00:00'
author: deaves
layout: post
permalink: /2010/11/configuring-netatalk-under-ubuntu/
categories:
    - 'Linux Admin'
tags:
    - linux
    - netatalk
    - ubuntu
---

```bash
sudo su -
cd /usr/src

apt-get source netatalk
apt-get install devscripts fakeroot libssl-dev cracklib2-dev
apt-get build-dep netatalk

cd netatalk-2*
DEB_BUILD_OPTIONS=ssl debuild
dpkg -i ../netatalk*.deb
echo "netatalk hold" | sudo dpkg --set-selections
```

**/etc/default/netatalk**:

```ini
AFPD_MAX_CLIENTS=20

ATALK_NAME=`/bin/hostname --short`

ATALK_MAC_CHARSET='MAC_ROMAN'
ATALK_UNIX_CHARSET='LOCALE'

AFPD_GUEST=nobody

ATALKD_RUN=no
PAPD_RUN=no
TIMELORD_RUN=no
A2BOOT_RUN=no
CNID_METAD_RUN=yes
AFPD_RUN=yes

ATALK_BGROUND=no

export ATALK_MAC_CHARSET
export ATALK_UNIX_CHARSET
```

**/etc/netatalk/AppleVolumes.default**:

```conf
~/ "$u" allow:user1,user2 options:usedots,upriv
/var/TimeMachine TimeMachine allow:user1,user2 options:usedots,upriv
```

- - - - - -

## Configuring avahi daemon

```bash
apt-get install avahi-daemon libnss-mdns
```

**/etc/nsswitch.conf**

```conf
hosts: files mdns4_minimal [NOTFOUND=return] dns mdns4 mdns
```

**/etc/avahi/services/afpd.service**

```xml
<?xml version="1.0" standalone=no?><!--*-nxml-*-->

<service-group>
<name replace-wildcards="yes">%h</name>
<service>
<type>_afpovertcp._tcp</type>
<port>548</port>
</service>
<service>
<type>_device-info._tcp</type>
<port>0</port>
<txt-record>model=Xserve</txt-record>
</service>
</service-group>
```

**Model Options**: *Xserve (same a RackMac) / PowerBook / PowerMac / Macmini / iMac / MacBook / MacBookPro / MacBookAir / MacPro / AppleTV1,1 / AirPort*  
**Ref**: */System/Library/CoreServices/CoreTypes.bundle/Contents/Info.plist*

- - - - - -

## Configuring TimeMachine

```conf
defaults write com.apple.systempreferences TMShowUnsupportedNetworkVolumes 1
```

**System Restore using TimeMachine**: From OSX boot dvd you will need to drop to a shell and manually mount the share in the /Volumes directory…

```bash
mount -t afp afp://username:password@hostname/TimeMachine /Volumes/TimeMachine
```

- - - - - -

**FYI**: There is no direct way to quota or limit share sizes on AFS shares, you must either configure your share on a separate partition, create a loopback filesystem for the share or manually create a sparcebundle file with a limit in DiskUtility (named “&lt;host&gt;\_&lt;mac&gt;.spacebundle”) and configure TimeMachine to backup to it.

```bash
dd if=/dev/zero of=/var/TimeMachine.img bs=10M count=10240
mkfs -t ext2 -b 2048 -v Timemachine.img
echo "/var/TimeMachine.img /mnt/TimeMachine auto loop,auto,rw   0       0" >> /etc/fstab
mkdir /mnt/TimeMachine
mount /mnt/TimeMachine
```
