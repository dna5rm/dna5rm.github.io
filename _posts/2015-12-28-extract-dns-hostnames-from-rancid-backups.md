---
title: 'Extract DNS Hostnames from rancid backups'
date: '2015-12-28T16:22:16+00:00'
author: deaves
layout: post
permalink: /2015/12/extract-dns-hostnames-from-rancid-backups/
categories:
    - Cisco
    - Linux
    - 'Linux Scripts'
tags:
    - cisco
    - dns
    - host
    - 'ip dns server'
    - rancid
---

```bash
## To be ran from against rancid configs directory ##
# 1st loop greps and out all interfaces from the config.
# sed sterilizes the output, converts to lowercase and shorten interfaces names.
# 2nd loop prints the output and excludes uninteresting lines.

for CONFIG in ~rancid/rancid/configs/*
 do grep -e ^"interface" -e ^" ip address" $CONFIG 2> /dev/null |\
  tr -d '[:cntrl:]' | sed 's/interface /\n/g' | grep "ip address [1-9]" | awk '{print $1,$4}' |\
  sed 's/\(.*\)/\L\1/;s/vlan/vl/;s/loopback/lo/;s/gigabitethernet/gi/;s/fastethernet/fa/;s/port-channel/po/;s/tunnel/tu/;s/serialf/se/;s/dialer/di/' |\
  awk '{print "'$(basename $CONFIG)'",$0}'
 done | while read HOST INTERFACE ADDRESS
  do INTERFACE=`echo $INTERFACE | sed 's/\//-/g;s/\./-/g;s/:/-/g'`
   [ "$(host "$INTERFACE.$HOST" | awk '{print $NF}')" != "$ADDRESS" ] && { printf "ip host $INTERFACE.$HOST $ADDRESS\n"; }
 done
```
