---
title: 'TCL/Expect script to backup Cisco device configs.'
date: '2017-03-31T08:09:40+00:00'
author: deaves
layout: post
permalink: /2017/03/expect-backup-cisco-device-configs/
categories:
    - Cisco
    - Linux
    - 'Linux Admin'
    - 'Linux Scripts'
    - Networking
tags:
    - bash
    - expect
    - linux
    - logrotate
    - rancid
    - rtrinfo
    - script
    - shell
    - tcl
---

I am not a software developer, but I do like challenges and am interested in learning about different software languages. For this project I decided to practice some TCL/Expect so I rewrote a poorly written Perl script I came across. This script will back up Cisco device configurations by reading 2 files: *command.db* and *device.db* ... It loads them into a data dictionary and iterates through it using a control loop. It then logs into the device, by shelling out to rancid, and executes all its commands. Its a little hacky, but works. I even wrote a shell script to parse the log output into separate files.

## /srv/rtrinfo/rtrinfo.exp: a expect script, ASCII text executable

```tcl
#!/usr/bin/expect -f
# Login to a list of devices and collect show output.
#
## Requires: clogin (rancid)

exp_version -exit 5.0
set timeout 5

set DEVDB "[lindex $argv 0]"
set LOGDIR "/var/log/rtrinfo"
set OUTLOG "/srv/rtrinfo/output.log"

## Validate input files or print usage.
if {0==[llength $DEVDB]} {
    send_user "usage: $argv0 -device.db-\n"
    exit
} else {
   if {[file isfile "cmd.db"] == "1"} {
      set CMDDB "cmd.db"
   } elseif {[file isfile "[file dirname $argv0]/cmd.db"] == "1"} {
      set CMDDB "[file dirname $argv0]/cmd.db"
   } else {
    send_user "Unable to find cmd.db file, can not start...\n"
    exit 1
   }
}

################################################################

### Procedure to create 3 column dictionary ###
proc addDICT {dbVar field1 field2 field3} {

   # Initialize the DEVICE dictionary
   if {![info exists $dbVar]} {
      dict set $dbVar ID 0
   }

   upvar 1 $dbVar db

   # Create a new ID
   dict incr db ID
   set id [dict get $db ID]

   # Add columns into dictionary
   dict set db $id "\"$field1\" \"$field2\" \"$field3\""
}

### Build the CMD and DEVICE dicts from db files ###
foreach DB [list $CMDDB $DEVDB] {
   set DBFILE [open $DB]
   set file [read $DBFILE]
   close $DBFILE

   ## Split into records on newlines
   set records [split $file "\n"]

   ## Load records for dictionary
   foreach rec $records {
      ## split into fields on colons
      set fields [split $rec ";"]
      lassign $fields field1 field2 field3

      if {"[file tail $DB]" == "cmd.db"} {
         # Cols: OUTPUT TYPE CMD
         foreach field2 [split $field2 ","] {
            addDICT CMDS $field2 $field1 $field3
         }
      } else {
         # Cols: HOST TYPE STATE DESC
         addDICT DEVICES $field1 $field2 $field3
      }
   }
}

################################################################

### Open $OUTLOG to be used for post parcing.
set OUTLOG [open "$OUTLOG" w 0664]

### Itterate the DEVICES dictionary ###
dict for {id row} $DEVICES {

   ## Assign field names
   lassign $row DEVICE DEVTYPE STATUS

   ## Process device status
   if {"$STATUS" == "up"} {

      ## Create log output directory if does not exist
      if {[file isdirectory "$LOGDIR"] != "1"} {
         file mkdir "$LOGDIR"
      }

      log_file
      log_file -noappend "$LOGDIR/$DEVTYPE\_$DEVICE.log"

      ## Run rancid's clogin with a 5min timeout.
      spawn timeout 300 clogin $DEVICE

      expect "*#" {

      ## Set proper terminal length ##
      if {$DEVTYPE != "asa"} {
         send "terminal length 0\r"
      } else {
         send "terminal pager 0\r"
      }

      ### Itterate the CMDS dictionary ###
      dict for {id row} $CMDS {
         ## Assign field names
         lassign $row CMDTYPE OUTPUT CMD

         ## Push commands to device & update $OUTLOG
         if {($DEVTYPE == $CMDTYPE)&&($OUTPUT != "")} {
            puts $OUTLOG "$LOGDIR/$DEVTYPE\_$DEVICE.log;$OUTPUT;$CMD"
            expect "*#" { send "$CMD\r" }
         }
      }

      ## We are done! logout
      expect "*#" { send "exit\r" }
      expect EOF
      }

   }
}

close $OUTLOG

### Run a shell script to parse the output.log ###
#exec "[file dirname $argv0]/rtrparse.sh"
```

## /srv/rtrinfo/cmd.db: ASCII text

```
acl;asa,router;show access-list
arp;ap,ace,asa,router,switch;show arp
arpinspection;ace;show arp inspection
arpstats;ace;show arp statistics
bgp;router;show ip bgp
bgpsumm;router;show ip bgp summary
boot;switch;show boot
cdpneighbors;ap,router,switch;show cdp neighbors
conferror;ace;sh ft config-error
controller;router;show controller
cpuhis;ap,router,switch;show process cpu history
debug;ap,router,switch;show debug
dot11ass;ap;show dot11 associations
envall;switch;show env all
env;router;show environment all
errdis;switch;show interface status err-disabled
filesys;router,switch;dir
flash;asa;show flashfs
intdesc;ap,router,switch;show interface description
interface;ap,asa,router,switch;show interface
intfbrie;ap,ace,router,switch;show ip interface brief
intipbrief;asa;show interface ip brief
intstatus;switch;show interface status
intsumm;router;show int summary
inventory;asa,router,switch;show inventory
iparp;ap,switch;show ip arp
ipint;router;show ip int
mac;switch;show mac address-table
nameif;asa;show nameif
ntpassoc;ap,asa,router,switch;show ntp assoc
plat;router;show platform
power;switch;show power inline
probe;ace;show probe
routes;asa;show route
routes;ap,router,switch;show ip route
rserver;ace;show rserver
running;ace;show running-config
running;ap,asa,router,switch;more system:running-config
serverfarm;ace;show serverfarm
service-policy;ace;show service-policy
service-pol-summ;ace;show service-policy summary
spantree;switch;show spanning-tree
srvfarmdetail;ace;show serverfarm detail
version;ap,ace,asa,router,switch;show version
vlan;switch;show vlan
```

## /srv/rtrinfo/device.db: ASCII text

```
192.168.0.1;router;up;Site Router
192.168.0.2;ap;up;Atonomous AP
192.168.0.3;asa;ASA Firewall
192.168.0.5;switch;Site Switch
192.168.0.10;ace;Cisco ACE
```

## /srv/rtrinfo/rtrparse.sh: Bourne-Again shell script, ASCII text executable

```bash
#!/bin/bash
# Parse the new rtrinfo output.log and create individual cmd output.
# 2016 (v.03) - Script from www.davideaves.com

OUTLOG="/srv/rtrinfo/output.log"
RTRPATH="$(dirname $OUTLOG)"

### Delete previous directories.
for DIR in ace asa router switch
 do [ -d "$RTRPATH/$DIR" ] && { rm -rf "$RTRPATH/$DIR"; }
done

### Itterate through $OUTLOG
grep "\.log" "$OUTLOG" | while IFS=';' read LOGFILE OUTPUT CMD
 do

 ### Get device name and type.
 TYPE="$(basename "$LOGFILE" | awk -F'_' '{print $1}')"
 DEVICE="$(basename "$LOGFILE" | awk -F'_' '{print $2}' | sed 's/\.log$//')"

 ### Create output directory.
 [ ! -d "$RTRPATH/$TYPE/$OUTPUT" ] && { mkdir -p "$RTRPATH/$TYPE/$OUTPUT"; }

 ### Extract rtrinfo:output logs and dump into individual files.
 # 1) sed identify $CMD output between prompts.
 # 2) awk drops X beginning line(s).
 # 3) sed to drop the last line.
 sed -n "/[^.].*[#, ]$CMD\(.\)\{1,2\}$/,/[^.].*#.*$/p" "$LOGFILE" \
 | awk 'NR > 0' | sed -n '$!p' > "$RTRPATH/$TYPE/$OUTPUT/$DEVICE.txt"

 ## EX: sed -n "/[^.]\([a-zA-Z]\)\{3\}[0-9].*[#, ]$CMD\(.\)\{1,2\}$/,/[^.]\([a-zA-Z]\)\{3\}[0-9].*#.*$/p"

done
```

Since this is something that would be collect nightly or weekly, I would probably kick this off using logrotate (as opposed to using crontab). The following would be what I would drop in my /etc/logrotate.d directory...

## /etc/logrotate.d/rtrinfo: ASCII text

```
/var/log/rtrinfo/*.log {
        rotate 14
        daily
        missingok
        compress
        sharedscripts
        postrotate
                /srv/rtrinfo/rtrinfo.exp /srv/rtrinfo/device.db > /dev/null
        endscript
}
```
