---
title: 'Traceroute script to detect route changes.'
date: '2016-01-05T15:53:35+00:00'
author: deaves
layout: post
permalink: /2016/01/traceroute-script-detect-route-changes/
categories:
    - Linux
    - 'Linux Scripts'
    - Networking
tags:
    - linux
    - mtr
    - 'route change'
    - script
---

The following script relies on [MTR](http://www.bitwizard.nl/mtr/) and is meant to be run in cron. It could be useful to log and/or detect route changes you the downstream provider path to multiple endpoint IP’s. Additionally the log-file is compressed using XZ tools so you do not have to worry about the logs growing to an unmanageable size very quickly.

```bash
#!/bin/bash
## Crontab Example: @hourly /opt/mtreport.sh -p

HOSTS="10.100.100.43 192.168.3.4 172.16.16.10"
LOGFILE="/srv/mtreport.log.xz"

# FUNCTION: End Script if error.
DIE() {
 echo "ERROR: Validate \"$_\" is installed and working on your system."
 exit 0
}

MTRRUN() {
 /usr/sbin/mtr --report --report-cycles 1 --raw --no-dns $HOST |\
  awk 'NR%2==1 {printf  " "$NF;} NR%2==0 {printf "|"$NF/1000;}'
}

# Validate script requirements are meet.
type -p /usr/sbin/mtr > /dev/null || DIE

if [ "$1" == "-p" ]; then

 # Main Loop.
 for HOST in $HOSTS
  do echo "$(date +%s)$(MTRRUN)" | xz -9 -c >> "$LOGFILE"
 done

elif [ ! -z "$1" ]; then

 xzgrep "$1" "$LOGFILE" | while read LINE
  do ARRAY=( $LINE )

   ## Show the Timestamp ##
   echo; date -d @${ARRAY[0]} +'%Y/%m/%d_%H:%M:%S'
   ARRAY=("${ARRAY[@]:1}") # Drop the timestamp array element

   ## Itirate through hops ##
   for HOP in "${ARRAY[@]}"
    do [ -z "$COUNT" ] && { COUNT=0; }
     echo "$COUNT|$HOP ms"
     let COUNT++ # Increment Hop Count
    done | column -ts\|
   done

else

 echo "Poll --> $0: -p"
 echo "View --> $0: x.x.x.x"

fi

```
