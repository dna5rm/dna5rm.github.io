---
title: 'Bulk clean corrupted MySQL tables under Linux'
date: '2015-11-18T16:52:58+00:00'
author: deaves
layout: post
permalink: /2015/11/bulk-clean-corrupted-mysql-tables-under-linux/
categories:
    - Linux
    - 'Linux Admin'
---

Recommend shutting down the MySQL running service: ```/etc/init.d/mysql stop```

To scan all DB files for errors: ```myisamchk -s /var/lib/mysql/*/*.MYI```

To fix a courted Database: ```myisamchk -r -update-state /var/lib/mysql/DIR/file.MYI```

To automate the scanning and fixing of all DB files: ```myisamchk -s /var/lib/mysql/*/*.MYI 2>&1 | grep ^MyISAM | awk '{print $2}' | sed "s/’//g" | sort | uniq | while read DBFILE; do myisamchk -r -update-state $DBFILE; done```
