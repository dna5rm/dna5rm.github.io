---
title: 'Cisco autonomous AP associated users script.'
date: '2015-01-19T16:50:11+00:00'
author: deaves
layout: post
permalink: /2015/01/cisco-autonomous-ap-associated-users-script/
categories:
    - Cisco
    - 'Linux Scripts'
    - Networking
tags:
    - cisco
    - linux
    - wireless
---

Apparently there is no SNMP string to query to get the number of users associated to each of your SSIDs. So I created a small script to connect to the AP via its web interface and pull down an associated user count. Eventually I’ll create a cacti template for this script. In the meanwhile its just standalone script.

```bash
#!/bin/bash
## Created by: deaves
# Query an autonomous Cisco AP and display a count of all users associated to each SSID.
#
## Requires: curl

# Required script variables.
APHOST="10.0.0.2"            # AP hostname
AUTHUP="joeuser:joepassword"        # USERNAME:PASSWORD

# FUNCTION: End Script if error.
DIE() {
 echo "ERROR: Validate \"$_\" is installed and working on your system."
 exit 0
}

# Validate script requirements are meet.
type -p curl > /dev/null || DIE


printf "%-8s %-20s %-5s\n" "Radio" "SSID" "Users"
printf "%-8s %-20s %-5s\n" "=======" "===================" "====="

# Main Loop.
curl --user ${AUTHUP} http://${APHOST}/ap_assoc.shtml 2> /dev/null | awk 'sub(/\"htmlClients\"/,""){f=1} /^">/{f=0} f' | awk 'NR > 2' | while read LINE
 do eval ARRAY=( $LINE )

  [ "${ARRAY[0]}" == "802.11" -a "${ARRAY[4]}" == "Dot11Radio0:" ] && { RADIO="2.4GHz" ;}
  [ "${ARRAY[0]}" == "802.11" -a "${ARRAY[4]}" == "Dot11Radio1:" ] && { RADIO="5GHz" ;}
  [ "${ARRAY[0]}" == "SSID" ] && { SSID="${ARRAY[1]}"; echo; unset MAC ;}
  [ "${ARRAY[6]}" == "Assoc" ] && { MAC+=( "${ARRAY[0]}" ) ;}

  [ -n "${SSID}" -a -n "${ARRAY[0]}" ] && { printf "${RADIO} ${SSID} ${#MAC[@]} " ;}

 done | awk '{print $1,$2,$NF}' | sed '1d;s/\[//g;s/\] / /g' | while read RADIO SSID ASSOC
 do

  printf "%-8s %-20s %-5s\n" "${RADIO}" "${SSID}" "${ASSOC}"

 done
```
