---
title: 'Script to do IPv6 DNS updates via DNSSEC.'
date: '2010-12-26T10:28:41+00:00'
author: deaves
layout: post
permalink: /2010/12/script-to-do-ipv6-dns-updates-via-dnssec/
categories:
    - 'Linux Scripts'
tags:
    - dns
    - ipv6
    - linux
    - script
    - updates
---

```bash
#!/bin/bash
## Created by: deaves
# IPv6 DNS update via DNSSEC.
#
## Requires: bind9utils, dnsutils

KEYFILE="<dnssec keyfile>"

### Server Vars ###
SERVER="<dns server>"
DOMAIN="<domain to update>"
TTL="3600"
PTRNET="0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.ip6.arpa"

### Host Vars ###
HOST="$(hostname -s)"
IPv6="$(/sbin/ifconfig | grep Global$ | sed 's/.*[^: ]*: //;s/\/.*//')"
PTR="$(nslookup ${IPv6} | sed 's/ip6.arpa.*/ip6.arpa/' | awk '{print $NF}' | grep arpa$)"

### Run nsupdate ###
nsupdate -v -k ${KEYFILE} 
```
