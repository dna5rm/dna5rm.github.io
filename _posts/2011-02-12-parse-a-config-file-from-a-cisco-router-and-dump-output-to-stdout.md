---
title: 'Parse a config file from a cisco router and dump output to stdout.'
date: '2011-02-12T12:11:07+00:00'
author: deaves
layout: post
permalink: /2011/02/parse-a-config-file-from-a-cisco-router-and-dump-output-to-stdout/
categories:
    - 'Linux Scripts'
    - Networking
tags:
    - cisco
    - config
    - parse
---

```bash
#!/bin/bash
## Created by: deaves
# Parse a config file from a cisco router and dump output to stdout.
# Output is tab seperated so it can be opened in a spreadsheet.
#
## Requires: ipcalc (http://jodies.de/ipcalc)

[ -z "$1" ] && { echo "Please give input"; exit 1;}

cat "$1" | grep -e ^"interface" -e ^" ip" -e ^" description" -e ^" service-policy" -e "!"$ | while read LINE; do
  eval ARRAY=( `echo '$LINE'` )

  ### INTERFACE ###
  if [ "${ARRAY[0]}" == "interface" ]; then
        INT[0]=`basename "$1"`
        INT[1]="${ARRAY[1]}"

        ### FR interface ###
        if [ "${ARRAY[2]}" == "point-to-point" ]; then
           INT[7]="${QOSsto}"
        else
           unset QOSsto
        fi

  ### INTERFACE DESCRIPTION ###
  elif [ "${ARRAY[0]}" == "description" ]; then
        INT[2]=`echo "${ARRAY[@]}" | awk -F'%' '{print $2}'`
        INT[3]=`echo "${ARRAY[@]}" | sed 's/description //;s/%.*//'`

  ### INTERFACE IP ###
  elif [ "${ARRAY[0]}" == "ip" ]; then

        ### ADDRESS ###
        if [ "${ARRAY[1]}" == "address" ]; then
           INT[5]=`ipcalc -nb ${ARRAY[2]} ${ARRAY[3]} | grep ^Network | awk '{print $NF}'`
           ROUTE=`ipcalc -nb ${ARRAY[2]} ${ARRAY[3]} | grep ^HostMax | awk '{print $NF}'`

           ### Static Routes ###
           INT[6]=`grep "${INT[1]} ${ROUTE}"$ "$1" | awk '{print $(NF-3),$(NF-2)}' | while read LINE; do
                ipcalc -nb ${LINE} | grep ^Network | awk '{print $NF" "}'
           done | tr -d '[:cntrl:]'`

        ### VRF ###
        elif [ "${ARRAY[1]}" == "vrf" ]; then
           INT[4]="${ARRAY[3]}"
        fi

  ### INTERFACE SERVICE-POLICY ###
  elif [ "${ARRAY[0]}" == "service-policy" ]; then
        # Using 2 variables to store QoS bit for FR interfaces.
        QOSsto="${ARRAY[2]}"
        INT[7]="${ARRAY[2]}"

  ### Print Result ###
  elif [ "${ARRAY[0]}" == "!" ] && [ -n "${INT[5]}" ]; then
        printf "\"${INT[0]}\"\t\"${INT[1]}\"\t\"${INT[2]}\"\t\"${INT[3]}\"\t\"${INT[4]}\"\t\"${INT[5]}\"\t\"${INT[6]}\"\t\"${INT[7]}\"\n"
        unset INT
  fi

done
```
