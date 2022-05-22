---
title: 'Parse all router configs in a directory and dump all DHCP pool settings to STDOUT.'
date: '2011-12-30T16:50:07+00:00'
author: deaves
layout: post
permalink: /2011/12/parse-all-router-configs-in-a-directory-and-dump-all-dhcp-pool-settings-to-stdout/
categories:
    - 'Linux Scripts'
    - Networking
tags:
    - cisco
    - dhcp
    - linux
    - parse
---

```bash
#!/bin/bash
## Created by: deaves
# Parse all router configs in a directory and dump all DHCP pool settings to STDOUT.
# Output will be a tab-separated XLS.
#
## Requires: ipcalc (http://jodies.de/ipcalc)

[ -z "$1" ] && { echo "Please give input"; exit 1;}

let COUNT=0

printf "hostname\tpool\tnetwork\tdomain-name\tdefault-router\tdns-server\tnetbios-name-server\tnetbios-node-type\tlease\tclient_host\tclient_identifier\toption_0\toption_1\toption_2\tdhcp_excludes\t\t\n"

for CONFIG in $1/*; do

 ### Get router hostname
 RTRHOST=`grep ^"hostname" "${CONFIG}" | awk '{print $NF}'`

 ### Collect variables
 cat "${CONFIG}" | sed 's\   \^\g;s\;\_\g' | while read LINE; do
  eval ARRAY=( `echo '$LINE'` )

  ### DHCP pool name and excludes
  if [ "${ARRAY[0]} ${ARRAY[1]} ${ARRAY[2]}" == "ip dhcp pool" ]; then
   echo "dPOOL=\"`echo $LINE | awk '{print $NF}'`\"" >> "/tmp/${UID}-${PPID}.tmp"
   echo "dEXCLUDES=\"`cat ${CONFIG} | grep ^"ip dhcp excluded-address" | sed 's/ /-/g;s/ip-dhcp-excluded-address-/, /' | tr -d '[:cntrl:]' | cut -d' ' -f2-`\"" >> "/tmp/${UID}-${PPID}.tmp"
  fi

  ### Pull out DHCP Variables
  [ "${ARRAY[0]}" == "^network" ] && { echo "dNETWORK=\"`ipcalc ${ARRAY[1]} ${ARRAY[2]} | grep ^Network | awk '{print $2}'`\"" >> "/tmp/${UID}-${PPID}.tmp" ;}
  [ "${ARRAY[0]}" == "^option" ] && { echo "dOPTION${COUNT}=\"`echo $LINE | cut -d' ' -f2-`\"" >> "/tmp/${UID}-${PPID}.tmp" ; let COUNT++ ;}
  [ "${ARRAY[0]}" == "^default-router" ] && { echo "dDEFAULTROUTER=\"`echo $LINE | cut -d' ' -f2-`\"" >> "/tmp/${UID}-${PPID}.tmp" ;}
  [ "${ARRAY[0]}" == "^dns-server" ] && { echo "dDNSSERVER=\"`echo $LINE | cut -d' ' -f2-`\"" >> "/tmp/${UID}-${PPID}.tmp" ;}
  [ "${ARRAY[0]}" == "^netbios-name-server" ] && { echo "dNBNAMESERVER=\"`echo $LINE | cut -d' ' -f2-`\"" >> "/tmp/${UID}-${PPID}.tmp" ;}
  [ "${ARRAY[0]}" == "^netbios-node-type" ] && { echo "dNBNODETYPE=\"`echo $LINE | cut -d' ' -f2-`\"" >> "/tmp/${UID}-${PPID}.tmp" ;}
  [ "${ARRAY[0]}" == "^domain-name" ] && { echo "dDOMAINNAME=\"`echo $LINE | cut -d' ' -f2-`\"" >> "/tmp/${UID}-${PPID}.tmp" ;}
  [ "${ARRAY[0]}" == "^lease" ] && { echo "dLEASE=\"`echo $LINE | cut -d' ' -f2-`\"" >> "/tmp/${UID}-${PPID}.tmp" ;}
  [ "${ARRAY[0]}" == "^host" ] && { echo "dCLIENTHOST=\"`echo ${ARRAY[1]}`\"" >> "/tmp/${UID}-${PPID}.tmp" ;}
  [ "${ARRAY[0]}" == "^client-identifier" ] && { echo "dCLIENTIDENT=\"`echo $LINE | cut -d' ' -f2-`\"" >> "/tmp/${UID}-${PPID}.tmp" ;}
  [ "${ARRAY[0]}" == "^hardware-address" ] && { echo "dCLIENTIDENT=\"`echo $LINE | cut -d' ' -f2-`\"" >> "/tmp/${UID}-${PPID}.tmp" ;}

  ### Done; dump scope to STDOUT
  if [ "${ARRAY[0]}" == "!" ] && [ -e "/tmp/${UID}-${PPID}.tmp" ]; then
   . "/tmp/${UID}-${PPID}.tmp"

   printf "${RTRHOST}\t${dPOOL}\t${dNETWORK}\t${dDOMAINNAME}\t${dDEFAULTROUTER}\t${dDNSSERVER}\t${dNBNAMESERVER}\t${dNBNODETYPE}\t${dLEASE}\t${dCLIENTHOST}\t${dCLIENTIDENT}\t${dOPTION0}\t${dOPTION1}\t${dOPTION2}\t${dEXCLUDES}\t\t\n"

   let COUNT=0
   unset dPOOL dNETWORK dDOMAINNAME dDEFAULTROUTER dDNSSERVER dNBNAMESERVER dNBNODETYPE dLEASE dCLIENTHOST dCLIENTIDENT dOPTION0 dOPTION1 dOPTION2 dEXCLUDES
   rm "/tmp/${UID}-${PPID}.tmp"
  fi

 done

 unset RTRHOST
done
```
