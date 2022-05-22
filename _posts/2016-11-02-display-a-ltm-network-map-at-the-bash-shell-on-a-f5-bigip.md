---
title: 'Display a LTM Network Map at the bash shell on a F5 BigIP.'
date: '2016-11-02T20:35:25+00:00'
author: deaves
layout: post
permalink: /2016/11/display-a-ltm-network-map-at-the-bash-shell-on-a-f5-bigip/
categories:
    - F5
    - Linux
    - 'Linux Scripts'
    - 'Load Balancing'
tags:
    - bigip
    - f5
    - ltm
    - 'network map'
    - script
---

I have been doing a bunch of F5 migrations lately and have gotten fond of the visualization of the network map in the F5 GUI. From the CLI to get the status of a VIP you have to parse tmsh output to find the information your looking for. As a personal challenge I wanted to make a script to be ran on the F5 to provide the exact same summary you see in the GUI, but instead form a bash shell...

## Network MAP from the GUI

![f5_ltm-map-gui](/assets/F5_LTM-MAP-GUI.png)

## Network Map from the F5 bash shell, using this script

![f5_ltm-map-cli](/assets/F5_LTM-MAP-CLI.png)

## MAP.sh

```bash
#!/bin/bash
## Display a LTM Network Map at the bash shell on a F5 BigIP.
## 2016 (v1.0) - Script from www.davideaves.com

DEBUG="N"

# Fix broken TERM variable if using screen.
[ "$TERM" == "screen-256color" ] && { export TERM="xterm-256color"; }

# Start: Collect time for runtime.
TIME=`date +%s`

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

DIE() {
 ## End Script if error.
 ERR=$?
 echo "$c1BOLD""ERROR"" [$cbRED$ERR$cbBLACK]:$cCLS An error has been encountered."
 exit 1;
}

READID() {
 ## Return component & identifier of an array element.
 [ -z "$ID" ] && { echo "READID: Varable \"\$ID\" not found."; exit 1; }
 [ -z "$MAP" ] && { echo "READID: Varable \"\$MAP\" not found."; exit 1; }

 export COMPONENT="$(echo ${MAP[$ID]} | awk -F':' '{printf $1}')"
 export IDENTIFIER="$(echo ${MAP[$ID]} | awk -F':' '{printf $2}')"
}

STATUS() {
 ## Display status back to the user.
 [ -z "$STATUS" ] && { set > t; echo "STATUS: Varable \"\$STATUS\" not found."; exit 1; }

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

# Validate script requirements are meet.
[ -f "/config/bigip.conf" ] && [ -x "/usr/bin/tmsh" ] || { DIE; }

### Main Loop ###
grep ^"ltm virtual .*"$1".* {$" "/config/bigip.conf" | awk '{print $(NF-1)}' | while read VS
 do ((ID=0))

  # Read VS into an array.
  MAP=( `tmsh show ltm virtual $VS detail | grep -e "Ltm::" -e "Availability" -e "State" | sed 's/|//g;s/ \+//g' | awk -F':' '{print $(NF-1)":"$NF}'` )

  # DEBUG #
  [ "$DEBUG" == "Y" ] && { echo "$cfBLACK$cbCYAN${MAP[*]}$cCLS"; echo; }

  # Display Virtual Server #
  STATUS="$(((ID=ID+1)); READID; printf "$IDENTIFIER") $(((ID=ID+2)); READID; printf "$IDENTIFIER")"

  READID
  printf -- "$(STATUS)"
  printf "$IDENTIFIER $(echo "${MAP[`expr ${#MAP[@]}-3`]}" | awk -F'(' '{print "("$NF}')\n"

  ### Iterate from right of MAP array ###
  ((ID=${#MAP[@]}))
  while [ "$ID" -ge "1" ]
   do ((ID--)); READID

     # Display iRules #
     [ "$IDENTIFIER" == "HTTP_REQUEST" ] && { printf "   (*) $COMPONENT\n"; }

  done

  ### Iterate from left of MAP array ###
  ((ID=2))
  while [ "$ID" -le "${#MAP[@]}" ]
   do ((ID++)); READID

     # Display Nodes #
     if [ "$COMPONENT" == "Node" ]
      then
        STATUS="$(((ID=ID-2)); READID; printf "$IDENTIFIER") $(((ID=ID-1)); READID; printf "$IDENTIFIER")"

        printf -- "      $(STATUS)"
        printf "$(((ID=ID-3)); READID; printf "$COMPONENT:$IDENTIFIER") "
        printf "$( echo $IDENTIFIER | awk -F'(' '{print "("$NF}')\n"

     # Display Pools #
     elif [ "$COMPONENT" == "Pool" ]
      then
        STATUS="$(((ID=ID+1)); READID; printf "$IDENTIFIER") $(((ID=ID+2)); READID; printf "$IDENTIFIER")"

        printf -- "   $(STATUS)"
        printf "$IDENTIFIER\n"
     fi

  done; echo

done 2> /dev/null || DIE

# Done: Print status & runtime.
printf "Completed in $B$(expr `date +%s` - $TIME)$B0 seconds...$CLS\n"
```
