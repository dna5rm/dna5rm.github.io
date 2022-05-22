---
title: 'Use transmission to download all torrents in the current directory'
date: '2010-12-08T05:18:57+00:00'
author: deaves
layout: post
permalink: /2010/12/use-transmission-to-download-all-torrents-in-the-current-directory/
categories:
    - 'Linux Admin'
tags:
    - bittorrent
    - linux
    - transmission
---

```bash
find ./ -type f -name "*.torrent" | while read FILE; do echo "Spooling: $FILE"; transmission-remote -w "$(dirname "$FILE")" -a "$FILE"; sleep 1; done
```
