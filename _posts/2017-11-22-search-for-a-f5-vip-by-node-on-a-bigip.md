---
title: 'Search for a F5 VIP by Node on a BigIP.'
date: '2017-11-22T16:29:34+00:00'
author: deaves
layout: post
permalink: /2017/11/search-for-a-f5-vip-by-node-on-a-bigip/
categories:
    - F5
    - Linux
    - 'Linux Scripts'
    - 'Load Balancing'
    - Networking
tags:
    - bigip
    - f5
    - irule
    - node
    - policy
    - script
    - tmsh
---

About a year ago I wrote two F5 scripts for [mapping](/2016/11/display-a-ltm-network-map-at-the-bash-shell-on-a-f5-bigip/) and [filtering](/2016/11/filter-out-a-single-f5-virtual-server-config-on-a-bigip/) VIPs on an F5 BigIP. I use those scripts often and the f5filter script I continue to tweak/correct minor bugs; which I post in the comments. The following script is cut from same cloth as the previous 2 scripts, except this script search for an F5 VIP by Node; even if the VIP directs traffic to the node via a Policy or iRule.

## f5node.sh: Bourne-Again shell script text executable

```bash
#!/bin/bash
## Reverse filter a single Node on a BigIP.
## 2017 (v1.0) - Script from www.davideaves.com

F5CONFIG="$1"
F5NODE="$2"

# Fix broken TERM variable if using screen.
[ "$TERM" == "screen-256color" ] && { export TERM="xterm-256color"; }

# FUNCTION: End Script if error.
DIE() {
 echo "ERROR: Validate \"$_\" is installed and working on your system."
 exit 0
}

# ANSI color variables.
esc="$(echo -e '\E')";  cCLS="${esc}[0m"
cfBLACK="${esc}[30m";   cbBLACK="${esc}[40m"
cfRED="${esc}[31m";     cbRED="${esc}[41m"
cfGREEN="${esc}[32m";   cbGREEN="${esc}[42m"
cfYELLOW="${esc}[33m";  cbYELLOW="${esc}[43m"
cfBLUE="${esc}[34m";    cbBLUE="${esc}[44m"
cfMAGENTA="${esc}[35m"; cbMAGENTA="${esc}[45m"
cfCYAN="${esc}[36m";    cbCYAN="${esc}[46m"
cfWHITE="${esc}[37m";   cbWHITE="${esc}[47m"
c1BOLD="${esc}[1m";     c0BOLD="${esc}[22m"

### Print Syntax if arguments are not provided. ###
if [ ! -e "$F5CONFIG" ] || [ -z "$F5NODE" ]
 then
 echo "Usage: $0 bigip.conf 10.1.1.10"
 exit 0;
fi

### Check to see if we are running this on an F5. ###
type -p tmsh > /dev/null && if [ "$(cat /var/prompt/ps1)" == "Active" ]
  then STATE=$cbGREEN
  else STATE=$cbRED
fi

### The function that does all the filtering. ###
F5FILTER() {
 if [[ "$(file "$F5CONFIG")" == *"ASCII"* ]]
  then cat "$F5CONFIG"
 elif [[ "$(file "$F5CONFIG")" == *"gzip compressed data"* ]]
  then tar -xOvf "$F5CONFIG" config/bigip.conf 2> /dev/null
 fi | sed -n -e '/^ltm '"$(echo $F5STANZA)"'.*/,/^}$/p' | sed 's/^}/}|/g' | tr -d '[:cntrl:]' |\
  sed 's/|/\n/g;s/ \{1,\}/ /g' | grep "$F5DIG"
}

### Display status icon back to the user. ###
STATUS() {
 if    [ "$STATUS" == "available enabled" ]; then printf -- "($c0BOLD$cfBLACK$cbGREEN@$cCLS) "
  elif [ "$STATUS" == "available disabled" ]; then printf -- "($c0BOLD$cfBLACK$cbBLUE@$cCLS) "
  elif [ "$STATUS" == "unknown enabled" ]; then printf -- "[$c0BOLD$cfBLACK$cbBLUE#$cCLS] "
  elif [ "$STATUS" == "offline disabled" ]; then printf -- " "
  elif [ "$STATUS" == "offline enabled" ]; then printf -- " "
  elif [ "$STATUS" == "available disabled-by-parent" ]; then printf -- "($c1BOLD$cfBLACK$cbWHITE@$cCLS) "
  elif [ "$STATUS" == "unknown disabled-by-parent" ]; then printf -- "[$c1BOLD$cfBLACK$cbWHITE#$cCLS] "
  elif [ "$STATUS" == "offline disabled-by-parent" ]; then printf -- " "
  else printf "[ $STATUS ] "
 fi
}

### Display the final result back to the user. ###
PRINT() {
 if [ ! -z "$STATE" ]
  then ### We are running this script on the F5. ###
   printf "$STATE$cfBLACK$(cat /var/prompt/hostname)$cCLS "

   # Collect status of VIRTUAL and print.
   printf ">>> "
   STATUS="$(tmsh show ltm virtual $VIRTUAL 2> /dev/null | grep -e "Availability" -e "State" | awk -F':' '{printf $NF" "}' | sed 's/ \+/ /g;s/^[ \t]*//;s/[ \t]*$//')"
   STATUS
   printf "$c1BOLD$VIRTUAL$cCLS  /dev/null | grep -e "Availability" -e "State" | awk -F':' '{printf $NF" "}' | sed 's/ \+/ /g;s/^[ \t]*//;s/[ \t]*$//')"
   STATUS
   printf "$cfCYAN$POOL$cCLS\n"

   # Collect status of Nodes and print.
   for NODE in $NODES;
    do STATUS="$(tmsh show ltm node $NODE 2> /dev/null | grep -e "Availability" -e "State" | awk -F':' '{printf $NF" "}' | sed 's/ \+/ /g;s/^[ \t]*//;s/[ \t]*$//')"
       STATUS && printf "$NODE "
   done | sed 's/\] /\]_/g' | xargs -n4 | column -t | awk '{print "  ",$0}' |\
   sed 's/'"$F5NODE"'/'"$cfYELLOW$F5NODE$cCLS "'/g;s/\]_/\] /g'

 else ### We are not Running on an F5 ###
   printf ">>> $c1BOLD$VIRTUAL$cCLS 
```
