---
title: 'Dialog wrapper for mpeg4ip to set the meta tags in a MP4 file.'
date: '2010-11-15T00:49:38+00:00'
author: deaves
layout: post
permalink: /2010/11/dialog-wrapper-for-mpeg4ip-to-set-the-meta-tags-in-a-mp4-file/
categories:
    - 'Linux Scripts'
tags:
    - dialog
    - linux
    - media
    - mpeg4ip
---

```bash
#!/bin/bash
## Created by: deaves
# Dialog wrapper for mpeg4ip to set the meta tags in a MP4 file.
#
## Requires: dialog, mpeg4ip

if [ -z "$1" ]; then
 printf "$0: Please specify a mp4 file.\n"
 exit 0;
else
 FILE="$1"
 opts=( s g a y A t T G d D w b c )
fi

loadTags () {
  mp4info "${FILE}" | grep ^" " | sed 's/\ //;s/\"/\\\\\"/g' | \
   while read A B
    do 
    if [ "${A}" == "Disk:" ]; then
     echo ${A} ${B}\" | sed 's/\:\ /=\"/;s/\ of\ /\"\;\ '${A}'/;s/\:/s=\"/'
    elif [ "${A}" == "Track:" ]; then
     echo ${A} ${B}\" | sed 's/\:\ /=\"/;s/\ of\ /\"\;\ '${A}'/;s/\:/s=\"/'
    else
     echo ${A} ${B}\" | sed 's/\:\ /=\"/'
    fi
  done | grep -v ^"Cover Art" > "/tmp/${USER}-${PPID}.tmp"

  . "/tmp/${USER}-${PPID}.tmp"
  rm -f "/tmp/${USER}-${PPID}.tmp"
}

diaString () {
  dialog --stdout --backtitle "${FILE}" --no-shadow --form "Updating Metadata..." 0 0 12 \
        "Name" 0 0 "${Name}" 2 0 41 255 "Genre" 0 21 "${Genre}" 0 27 15 255 \
        "Artist/Director" 3 0 "${Artist}" 4 0 30 255 \
        "Year" 3 33 "${Year}" 4 33 9 4 \
        "Album" 5 0 "${Album}" 6 0 30 255 \
        "Track" 5 33 "${Track}" 6 33 4 3 "Tracks" 5 33 "${Tracks}" 6 38 4 3 \
        "Grouping" 7 0 "${Grouping}" 8 0 30 255 \
        "Disk" 7 33 "${Disk}" 8 33 4 3 "Disks" 7 33 "${Disks}" 8 38 4 3 \
        "Writer" 9 0 "${Writer}" 10 0 30 255 "Tempo" 9 33 "${Tempo}" 10 33 9 8 \
        "Comment" 11 0 "${Comment}" 12 0 41 255
}

loadTags; let count=0
diaString | while read LINE; do
 if [ "${LINE}" -ge "0" ]; then
   mp4tags -${opts[${count}]} ${LINE} "${FILE}"
 elif [ ! -z "${LINE}" ]; then
   mp4tags -${opts[${count}]} "${LINE}" "${FILE}"
 else
   mp4tags -r ${opts[${count}]} "${FILE}"
 fi 2> /dev/null
 let count++
done
```
