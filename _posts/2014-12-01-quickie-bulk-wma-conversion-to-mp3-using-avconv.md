---
title: 'Quickie: Bulk WMA conversion to MP3 using avconv.'
date: '2014-12-01T03:34:54+00:00'
author: deaves
layout: post
permalink: /2014/12/quickie-bulk-wma-conversion-to-mp3-using-avconv/
categories:
    - Linux
    - 'Linux Admin'
    - OSX
tags:
    - avconf
    - conversion
    - linux
    - media
---

I was recently playing around with MacOSX’s built-in dictation tools and had to convert a bunch of WMA files to a format that could be opened using Audacity.

The following one-liner uses a **for** loop to quickly convert each *.WMA* in the current working directory to a *.MP3* file using **avconv**. If your using an older package repository avconv could be substituted for ffmpeg.

```bash
for FILE in *.WMA;
 do FILE=`echo $FILE | sed 's/.WMA//'`;
  avconv -i $FILE.WMA -acodec libmp3lame -ab 128k $FILE.mp3 ;
done
```

Remember: *File names, including extensions, are case sensitive in Linux/Unix. Only files ending in “.WMA” will be iterated.*
