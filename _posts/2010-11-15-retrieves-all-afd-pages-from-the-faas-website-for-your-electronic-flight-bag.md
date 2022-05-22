---
title: "Retrieves all AFD pages from the FAA's website for your electronic flight bag."
date: '2010-11-15T23:39:49+00:00'
author: deaves
layout: post
permalink: /2010/11/retrieves-all-afd-pages-from-the-faas-website-for-your-electronic-flight-bag/
categories:
    - 'Linux Scripts'
tags:
    - afd
    - faa
    - linux
    - script
---

```bash
### Created by: deaves
# Retrieves all AFD pages from the FAA's website.
#
## Requires: wget, pdftk

REGIONS="al ec nc ne nw pac se sw"
DATE="18NOV2010"

sAFD () {
  ### Create Directory for the region ###
  [ ! -d "${REGION}" ] && { mkdir "${REGION}"; }

  ## Fetch the AFD Legend ##
  if [ ! -e "${REGION}/${REGION}_front_${DATE}.pdf" ]; then
    wget -q -P "${REGION}" "http://naco.faa.gov/pdfs/${REGION}_front_${DATE}.pdf"
    printf "${REGION}/${REGION}_front_${DATE}.pdf "
  else
    printf "${REGION}/${REGION}_front_${DATE}.pdf "
  fi

  ## Fetch the pages for the AFD ##
  for PAGE in {1..1000}
  do

    if [ ! -e "${REGION}/${REGION}_${PAGE}_${DATE}.pdf" ]; then
      wget -q -P "${REGION}" "http://naco.faa.gov/pdfs/${REGION}_${PAGE}_${DATE}.pdf" || break
      printf "${REGION}/${REGION}_${PAGE}_${DATE}.pdf "
    else
      printf "${REGION}/${REGION}_${PAGE}_${DATE}.pdf "
    fi

  done

  ## Fetch the AFD Supplemental ##
  if [ ! -e "${REGION}/${REGION}_rear_${DATE}.pdf" ]; then
    wget -q -P "${REGION}" "http://naco.faa.gov/pdfs/${REGION}_rear_${DATE}.pdf"
    printf "${REGION}/${REGION}_rear_${DATE}.pdf "
  else
    printf "${REGION}/${REGION}_rear_${DATE}.pdf "
  fi
}

for REGION in $REGIONS; do
  echo "Building AFD: ${REGION}_${DATE}.pdf"
  pdftk `sAFD` output "${REGION}_${DATE}.pdf"
done
```
