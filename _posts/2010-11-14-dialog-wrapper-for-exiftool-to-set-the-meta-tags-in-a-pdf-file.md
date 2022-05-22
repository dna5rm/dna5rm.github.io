---
title: 'Dialog wrapper for exiftool to set the meta tags in a PDF file.'
date: '2010-11-14T16:53:52+00:00'
author: deaves
layout: post
permalink: /2010/11/dialog-wrapper-for-exiftool-to-set-the-meta-tags-in-a-pdf-file/
categories:
    - 'Linux Scripts'
tags:
    - dialog
    - exiftool
    - linux
    - script
---

```bash
#!/bin/bash
## Created by: deaves
# Dialog wrapper for exiftool to set the meta tags in a PDF file.
#
## Requires: dialog, libimage-exiftool-perl

if [ -z "$1" ]; then
 printf "$0: Please specify a pdf file.\n"
 exit 0;
else
 FILE="$1"
 opts=( Title Subject Author Creator Producer Keywords ModifyDate CreateDate )
fi

loadTags () {
  exiftool "${FILE}" | sed 's/\ //' | while read TAG X FIELD
    do 

     if [ "${X}" == ":" ]; then
       echo "${TAG}=\"${FIELD}\""
     fi

  done > "/tmp/${USER}-${PPID}.tmp"

  . "/tmp/${USER}-${PPID}.tmp"
  rm -f "/tmp/${USER}-${PPID}.tmp"
}

diaString () {
  dialog --stdout --backtitle "${FILE}" --no-shadow --form "Updating Metadata..." 0 0 0 \
        "Title" 1 0 "${Title}" 1 10 49 255 \
        "Subject" 2 0 "${Subject}" 2 10 49 255 \
        "Author" 3 0 "${Author}" 3 10 49 255 \
        "Creator" 4 0 "${Creator}" 4 10 49 255 \
        "Producer" 5 0 "${Producer}" 5 10 49 255 \
        "Keywords" 6 0 "${Keywords}" 6 10 49 255 \
        "Modified" 7 24 "${ModifyDate}" 7 33 26 26 \
        "Created" 8 25 "${CreateDate}" 8 33 26 26
}

loadTags; let count=0
diaString | while read LINE; do

 ## Flush all existing meta tags...
 if [ "${count}" -eq "0" ]; then
   exiftool -*= "${FILE}" > /dev/null
 fi

 exiftool -${opts[${count}]}="${LINE}" "${FILE}" > /dev/null

 let count++

done
```
