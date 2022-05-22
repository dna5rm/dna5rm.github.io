---
title: 'Sort an eBook and set meta tags in the file.'
date: '2010-11-14T16:52:11+00:00'
author: deaves
layout: post
permalink: /2010/11/sort-an-ebook-and-set-meta-tags-in-the-file/
categories:
    - 'Linux Scripts'
tags:
    - ebook
    - exiftool
    - linux
    - 'meta data'
---

```bash
#!/bin/bash
## Created by: deaves
# Sort an eBook and set meta tags in the file.
#
## Requires: digISBN.sh, libimage-exiftool-perl

QUERY="$1"
FILE="$2"
MODIFYDATE="$(date +%Y:%m:%d\ %T)"

ANSI () {
  ### Setting the ANSI colors to be used as variables...
  esc="$(echo -e '\E')"

  CYAN="${esc}[36m"; RED="${esc}[31m"
  B="${esc}[1m"; B0="${esc}[22m"
  CLS="${esc}[0m"
}; ANSI

### Begin Script...
if [ -z $2 ]; then
 printf "eBook Sorting...\n\nSyntax:\t${B}$0${B0} \"<isbn>\" \"<file>\"${CLS}\n\n"
 exit 0;
fi 2> /dev/null

### Find Book Variables...
digISBN.sh "${QUERY}" > "/tmp/${USER}-${PPID}"

### Load Book Variables...
. "/tmp/${USER}-${PPID}" 2> /dev/null

if [ ! -z "${TITLE}" ]; then
  printf "Sorting: ${CYAN}${FILE}${CLS} - ${B}${TITLE}${CLS}\n"

  DATE="$(date -d "$(echo ${DATE} | awk -F\- '{print $1"-"$2"-"$3}' | sed 's/--$/-1-/;s/-$/-1/')" +"%Y:%m:%d\ %T")"

  ### Updating the metadata...
  exiftool -*= "${FILE}" > /dev/null && \
  exiftool -Title="${TITLE}" -Author="${CREATOR}" -Creator="${PUBLISHER}" -Keywords="${IDENTIFIER}" -Subject="${SUBJECT}" -ModifyDate="${MODIFYDATE}" -CreateDate="${DATE}" "${FILE}" > /dev/null
  ### Title Subject Author Creator Producer Keywords ModifyDate CreateDate ###

  #!! What the file is to be renamed to !!#
  ISBN="$(echo ${IDENTIFIER} | awk -F, '{print $1}' | awk -F: '{print $2}')"
  YEAR="$(echo ${DATE} | awk -F\: '{print $1}')"
  extFILE="$(echo ${FILE} | awk -F. '{print $NF}')"
  newFILE="$(echo ${TITLE} | awk -F'[:;]' '{print $1}' | sed 's/\//_/g;s/'\''//g' | cut -c 1-200) (${ISBN}).${extFILE}"
fi

[ -z "${SUBJECT}" ] && { SUBJECT="."; }

mkdir -p "${SUBJECT}"
chmod 644 "${FILE}"
mv "${FILE}" "${SUBJECT}/${newFILE}"

### Cleanup ###
unset $(cat "/tmp/${USER}-${PPID}" | awk -F= '{print $1" "}' | tr -d [:cntrl:]) 2> /dev/null
rm "/tmp/${USER}-${PPID}"

unset extFILE newFILE MODIFYDATE YEAR</file></isbn>
```
