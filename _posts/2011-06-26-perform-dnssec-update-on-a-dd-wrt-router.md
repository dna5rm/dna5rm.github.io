---
title: 'Perform DNSSEC update on a DD-WRT router.'
date: '2011-06-26T01:08:45+00:00'
author: deaves
layout: post
permalink: /2011/06/perform-dnssec-update-on-a-dd-wrt-router/
categories:
    - 'Linux Scripts'
    - Networking
tags:
    - dd-wrt
    - linux
    - script
---

```bash
#!/bin/sh
## Created by: deaves
### Perform DNSSEC update on a DD-WRT router. ###
# This script will install bind-client and bind-tools if not already installed.
# Under normal use this script will create an additional nvram variable "wan_ipaddr_old",
# this variable is used to prevent updating the bind server if the WAN IP has not changed.
#
# CRON: */15 * * * * root DNSupdate.sh
#
## Requires: DD-WRT v24-sp2 (mega) with 3M /jffs

## Log last run - Debug.
#date > "/jffs/tmp/nsupdate.run"

SERVER=""
DOMAIN=""
KEYFILE="/jffs/"

### Run nsupdate ###
if [ -s "/jffs/usr/bin/nsupdate" ]; then

 # Getting required nvram vars.
 router_name="$(nvram get router_name)"
 wan_ipaddr="$(nvram get wan_ipaddr)"
 wan_ipaddr_old="$(nvram get wan_ipaddr_old)"

 # Only perform nsupdate if ip has changed and wan_ipaddr is valid.
 if [ "${wan_ipaddr}" != "${wan_ipaddr_old}" ] && [ "${wan_ipaddr}" != "0.0.0.0" ]; then

/jffs/usr/bin/nsupdate << _EOF_
  server ${SERVER}
  key `awk '{print $1,$NF}' ${KEYFILE}`
  update delete ${router_name}.${DOMAIN} A
  update add ${router_name}.${DOMAIN} 3600 A ${wan_ipaddr}
  send
_EOF_

  nvram set wan_ipaddr_old="${wan_ipaddr}"
 fi

### Install requiured tools if needed. Requires 3M available. ###
elif [ ! -s "/jffs/usr/bin/nsupdate" ] && [ "$(df | grep "jffs"$ | awk '{print $4}')" -ge "3000" ]; then

 cd /jffs; mkdir -p /jffs/tmp/ipkg

 # Fetch and install each required package.
 for PKG in libgcc_4.1.2-14.3_mipsel.ipk uclibc_0.9.29-14.3_mipsel.ipk libopenssl_0.9.8i-3.2_mipsel.ipk bind-libs_9.5.0-P1-1.1_mipsel.ipk bind-client_9.5.0-P1-1.1_mipsel.ipk bind-tools_9.5.0-P1-1.1_mipsel.ipk; do

  printf "\rDownloading: ${PKG}     "
  [ ! -e "/jffs/${PKG}" ] && { wget "http://downloads.openwrt.org/kamikaze/8.09.2/brcm47xx/packages/${PKG}" &> /dev/null || printf " [ERROR]\n" ;}

  printf "\rInstalling: ${PKG}     "
  [ -e "/jffs/${PKG}" ] && { ipkg install "/jffs/${PKG}" &> /dev/null && rm "/jffs/${PKG}" || printf " [ERROR]\n" ;}

 done

 printf "\rFinished installing bind-client and bind-tools.          \n"

### Notify user if at a loss ###
else
 printf "Not enough free space to install bind-client and bind tools.\n"
fi

### Done ###
```
