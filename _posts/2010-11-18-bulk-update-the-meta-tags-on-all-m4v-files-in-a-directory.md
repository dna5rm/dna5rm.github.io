---
title: 'Bulk update the meta tags on all m4v files in a directory.'
date: '2010-11-18T00:19:37+00:00'
author: deaves
layout: post
permalink: /2010/11/bulk-update-the-meta-tags-on-all-m4v-files-in-a-directory/
categories:
    - 'Linux Scripts'
tags:
    - 'bulk update'
    - linux
    - 'meta tags'
    - script
---

```bash
#!/bin/bash
## Created by: deaves
# Bulk update the meta tags on all m4v files in a directory.
#
## Requires: mpeg4ip

DATA="EPISODES"

ls *.m4v | while read FILE; do

  FILE="$(echo "$FILE" | awk -F"." '{print $1}')"
  TRACKS="$(wc -l EPISODES | awk '{print $1}')"

  grep ^"$FILE" "${DATA}" | while IFS='|' read TRACK TITLE DIRECTOR WRITER YEAR COMMENT; do
        echo "$FILE 
  done

done
```
