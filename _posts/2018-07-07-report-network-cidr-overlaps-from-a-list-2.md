---
title: 'Report network CIDR overlaps from a list.'
date: '2018-07-07T18:07:03+00:00'
author: deaves
layout: post
permalink: /2018/07/report-network-cidr-overlaps-from-a-list-2/
categories:
    - AWK
    - 'Linux Scripts'
    - Networking
tags:
    - address
    - awk
    - cidr
    - network
    - overlap
---

I created the following AWK script to report on CIDR overlaps in Nexus ACL’s. It works by reading a list of addresses in CIDR notation and iterates through each block while adding each address to an array. If a /32 host is observed its added to a separate array for processing at the END of the script after the network table is built. The script is quick and makes locating unneeded lines in large ACL’s that are too large to really go over by hand. Some of the functions were re-purposed from another AWK script: [Determining IP address range from Subnet/CIDR](http://linuxcalling.blogspot.com/2014/01/determining-ip-address-range-from.html) from a blog called linuxcalling. There is a lot of other really nice code up there and I consider the original script worth checking out.

The following examples are how the script can be ran…

### Using echo or printf stdout

```bash
echo "192.168.0.0/23
192.168.0.0/30
192.168.1.22/32" | ./acloverlap.awk
OVERLAP: 192.168.0.0/30     HINT: 192.168.0.0
OVERLAP: 192.168.1.22/32   

# Hosts: 513
```

### List in a text file

```bash
./acloverlap.awk cidrlist.txt 
OVERLAP: 192.168.0.0/30     HINT: 192.168.0.0
OVERLAP: 192.168.1.22/32   

# Hosts: 513
```

### From clogin against a live nexus

```bash
clogin -c "show ip access-lists WCCP-ACL" NEXUS | awk '/permit/{print $4}' | acloverlap.awk
```

## acloverlap.awk: awk script text executable, ASCII text

```awk
#!/usr/bin/awk -f
## Report network CIDR overlaps from a list.
## 2018 (v.01) - Script from www.davideaves.com

# Convert IP to Interger.
function ip_to_int(input) {
  split(input, oc, ".")
  ip_int=(oc[1]*(256^3))+(oc[2]*(256^2))+(oc[3]*(256))+(oc[4])
  return ip_int
}

# Convert Interger to IP.
function int_to_ip(input) {
  str=""
  num=input
  for(i=3;i>=0;i--){
    octet = int (num / (256 ^ i))
    str= i==0?str octet:str octet "."
    num -= (octet * 256 ^ i)
  }
  return str
}

## MAIN: Build HOST arrays
//{
  gsub(/\r/, "") # Strip CTRL+M
  split($1, ADDR, "/")

  # line is a CIDR block.
  if(ADDR[2] != "32") {
    NETWORK=ip_to_int(ADDR[1])
    BROADCAST=(NETWORK + (2^(32-ADDR[2]) - 1))

    # Iterate NETWORK until BROADCAST
    COUNT=NETWORK
    while(BROADCAST >= COUNT) {
      if(HOST[int_to_ip(COUNT)] != int_to_ip(COUNT)) {
        HOST[int_to_ip(COUNT)]=int_to_ip(COUNT)
      } else {
        if(PREVIOUS != $1) {
          printf "OVERLAP: %-18s HINT: %s\n", $1, HOST[int_to_ip(COUNT)]
          PREVIOUS=$1
        }
      }
      COUNT+=1
    }
  } else {
    # Store /32 host for END
    HOST32[key]=ADDR[1]
    key+=1
  }
}

## Done building arrays.
## Scan for /32 host overlaps

END{
  for(key in HOST32) {
    printf "OVERLAP: %-18s\n", HOST32[key]"/32"
  }
  printf "\n# Hosts: %s\n", length(HOST) + length(HOST32)
}
```
