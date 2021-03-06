---
title: 'Quickly convert a file to MP4 using ffmpeg.'
date: '2010-11-16T00:46:13+00:00'
author: deaves
layout: post
permalink: /2010/11/quickly-convert-a-file-to-mp4-using-ffmpeg/
categories:
    - 'Linux Scripts'
tags:
    - convert
    - ffmpeg
    - linux
    - media
---

```bash
#!/bin/bash
## Created by deaves
# Quickly convert a file to MP4 using ffmpeg.
#
## Requires: ffmpeg (Ref: http://ubuntuforums.org/showthread.php?t=786095)

function usage () {
  ### Display the script arguments...
  printf "Convert a file to MP4 using ffmpeg (vcodec:H264 / acodec:AAC)\n\n"
  printf "Usage: $0 -K <kbps> <videofile>\n"
}

if [ "$1" == "-K" -a "$2" -gt "100" ]; then
  let BIT_RATE="$2 * 1024"
else
  usage && exit 1;
fi

if [ ! -f "$3.mp4" ]; then

  ### Two Pass; slow ffpreset.
  printf "$3 > 1st pass "
  ffmpeg -i "$3" -an -pass 1 -vcodec libx264 -vpre slow_firstpass -b $BIT_RATE -bt $BIT_RATE -threads 0 "$3.mp4" 2> /dev/null
  printf "> 2nd pass "
  ffmpeg -i "$3" -y -acodec libfaac -ab 128k -pass 2 -vcodec libx264 -vpre slow -b $BIT_RATE -bt $BIT_RATE -threads 0 "$3.mp4" 2> /dev/null
  printf "> $3.mp4\n Duration:"


  ### Cleanup.
  rm "ffmpeg2pass-0.log" "x264_2pass.log" "x264_2pass.log.mbtree"
  ffmpeg -i "$3.mp4" 2>&1 | grep -e Duration -e Stream | awk -F": " '{$1=""; print $0}'

else

  printf "$3\n Duration:"
  ffmpeg -i "$3.mp4" 2>&1 | grep -e Duration -e Stream | awk -F": " '{$1=""; print $0}'

fi
```
