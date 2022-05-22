---
title: 'Quickly update or delete a zonefile host entry.'
date: '2012-01-06T23:16:00+00:00'
author: deaves
layout: post
permalink: /2012/01/quickly-update-or-delete-a-zonefile-host-entry/
categories:
    - 'Linux Scripts'
tags:
    - bind
    - linux
    - update
---

```bash
#!/bin/bash
## Created By: deaves
# Quickly update or delete a zonefile host entry.
#
## Requires: dnsutils, netcat, sshfp (python script)

DOMAIN="foo.com"
SERVER="ns.foo.com"

##### Begin Script #####

function DELETE () {
### Purge DNS entry for HOST ###
nsupdate  /dev/null && sshfp -s $IPv4 2> /dev/null | awk '{print "update add '$HOST.$DOMAIN' 3600",$2,$3,$4,$5,$6}'`
  send
_EOF_
}

function usage () {
  ### Display the script arguments.
  printf "Usage: $0 [-du] -h <host> -i <ip>\n\n"
  printf "Requires one option!\n"
  printf "\t-d: Delete DNS <host> A and SSHFP records\n"
  printf "\t-u: Update DNS </host><host> A and SSHFP records\n\n"
}

while getopts "duh:i:" ARG; do
  case "${ARG}" in
    d) [ -z $ACTION ] && { ACTION="D"; };;
    u) [ -z $ACTION ] && { ACTION="U"; };;
    h) HOST="$OPTARG";;
    i) IPv4="$OPTARG";;
    ?) echo "Invalid option -$OPTARG"; exit 1;;
  esac
done 2> /dev/null

[ -n "$ACTION" ] && { echo "$ACTION: $HOST ($IPv4)" ; }

if [ "$ACTION" == "U" ]; then
  [ -z "$IPv4" ] && { echo "Error: Missing IP" && exit 1; }
  DELETE
  UPDATE
elif [ "$ACTION" == "D" ]; then
  [ -z "$HOST" ] && { echo "Error: Missing HOST" && exit 1; }
  DELETE
else
  usage && exit 1;
fi
```
