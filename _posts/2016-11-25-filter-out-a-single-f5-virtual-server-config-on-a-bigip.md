---
title: 'Filter out a single F5 virtual server config on a BigIP.'
date: '2016-11-25T20:26:59+00:00'
author: deaves
layout: post
permalink: /2016/11/filter-out-a-single-f5-virtual-server-config-on-a-bigip/
categories:
    - F5
    - Linux
    - 'Linux Scripts'
    - 'Load Balancing'
    - Networking
tags:
    - ace
    - DR
    - f5
    - filter
    - ltm
    - script
    - VIP
---

Over the last two years I have been involved in several successful migrations off of the Cisco ACE platform. While I do not consider myself an ACE expert I have discovered a nice filter option, albeit poorly documented, to display/isolate only the relevant configuration of a particular service-policy (EX: ***show running-config filter vip.example.com***). This filter option is great for both migrations and copying configurations from one ACE to another; for when you need an identical DR VIP. Because of the general usefulness of being to quickly isolate parts of a configuration, I created a script to do the same for the F5. Just like the filter option on the ACE, this script will parse through an existing bigip.conf for a virtual server and display only required configuration items: virtual-address, pool, node, monitor, policy, profile & rules.

## f5filter.sh

```bash
#!/bin/bash
## Filter out a single F5 virtual server config on a BigIP.
## 2016 (v1.0) - Script from www.davideaves.com

F5CONFIG="$1"
F5STANZA="$2"

### Print Syntax if arguments are not provided. ###
if [ ! -e "$F5CONFIG" ] || [ -z "$F5STANZA" ]
 then
 echo "Usage: $0 bigip.conf example.domain.com_80_vs"
 exit 0;
fi

### The function that does all the filtering. ###
F5FILTER() {
 sed -n -e '/^ltm .*'"$(echo $F5STANZA | sed 's/\//\\\//g')"' {$/,/^}$/ p' $F5CONFIG
}

### Build Search commands to run after loop finishes ###
F5FILTER "$F5CONFIG" "$F5STANZA" | while read A B C D
 do

  ### Stanza: policy, profile, rule
  if [ -n "LCOUNT" -a "$(echo $A | cut -c1)" == "/" ]
   then echo "$LCOUNT|$A" | grep -v ":[0-9]"
        let LCOUNT++

  ### Stanza: virtual server ###
  elif [ "$A" == "ltm" -a "$B" == "virtual" ]
   then echo "80|$B $C"

  ### Stanza: pool ###
  elif [ "$A" == "pool" ]
   then F5STANZA="$(echo $B | awk -F'/' '{print $NF}')"
   echo "70|$A $B"

   # Dig inside of pool stanza #
   F5FILTER "$F5STANZA" | while read A B C D
    do  if [ "$A" == "monitor" ]
         then echo "40|$B"
        elif [ "$(echo $A | cut -c1)" == "/" -a "$B" == "{" ]
         then echo "50|node $A" | grep ":[0-9]$" | awk -F':' '{print $1}'
        fi
   done

  ### Stanza: virtual address ###
  elif [ "$A" == "destination" ]
   then echo "90|virtual-address $(echo $B | awk -F':' '{print $1}')"

  ### Stanza: LOOP ###
  elif [ "$B" == "{" -a -z "$C" ]
   then LCOUNT="10"
   [ "$A" == "policies" ] && { LCOUNT="20"; }
   [ "$A" == "rules" ] && { LCOUNT="30"; }
  fi

done | sort -n | uniq | while IFS="|" read SEQ F5STANZA
 do printf "#%.0s" {1..60}
    printf "\r### $SEQ: $F5STANZA \n"
    F5FILTER "$F5STANZA"
done
```

There are a few limitations with this script... It will not pull out any objects referenced in policies or irules. It will also not pull out inherited profiles.
