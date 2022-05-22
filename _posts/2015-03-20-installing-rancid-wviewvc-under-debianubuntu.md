---
title: 'Installing Rancid w/ViewVC under Debian/Ubuntu'
date: '2015-03-20T15:28:21+00:00'
author: deaves
layout: post
permalink: /2015/03/installing-rancid-wviewvc-under-debianubuntu/
categories:
    - Cisco
    - Linux
    - 'Linux Admin'
    - Networking
tags:
    - backup
    - cisco
    - debian
    - linux
    - rancid
    - ubuntu
    - viewvc
---

When managing a network there are tools out there like Solarwinds, Cisco NCS or even Cisco Works that allow engineers to backup configurations; but tools like them can sometimes be unwieldy and/or clunky at their best. In the enterprise I general run ARCHIVE on Cisco routers and switches to automatically back up my configs to a Linux TFTP server. For other devices like the ASA and non-cisco gear, that do not have ARCHIVE functionality, I use a tool called [RANCID](http://www.shrubbery.net/rancid/ "RANCID") and [ViewVC](http://www.viewvc.org/ "ViewVC") for config backup and change tracking purposes.  

There is a lot of documentation on the internet about setting up rancid, so the following is a very condensed step-by-step on getting Rancid up and running on a Debian distro.

# Package Installation

```
sudo apt-get install rancid viewvc
```

# Configure Rancid

## Set the CVS folder

```
sudo pico /etc/rancid/rancid.conf
```

Add ***LIST_OF_GROUPS=”**rancid**“*** to the config file.

```
sudo -u rancid /usr/lib/rancid/bin/rancid-cvs
```

## Configure /etc/cron.d/rancid

```
sudo pico /etc/cron.d/rancid
```

```
MAILTO=root

# Run config differ hourly
0 23 * * *  rancid /usr/lib/rancid/bin/rancid-run

# Clean out config differ logs
50 23 * * * rancid /usr/bin/find /var/log/rancid -type f -mtime +2 -exec rm {} \;
```

## Configure ~rancid/.cloginrc

sudo -u rancid pico ~/.cloginrc && chmod 600 ~/.cloginrc

```
add user        *       rancid
add password    *       RANCIDPW RANCIDPW
add method      *       ssh telnet
```

## Configure device database

```
sudo -u rancid pico /var/lib/rancid/rancid/router.db
```

> Example: *DEXTER-CS01;cisco;up;Dexters Lab Core Switch*

Remember! For IPv6 compatibility RANCID now uses semicolons as a deliminator.

## Create Symbolic Lync for clogin

```
sudo ln -s /usr/lib/rancid/bin/clogin /usr/local/bin/clogin
```

# Configure ViewVC
