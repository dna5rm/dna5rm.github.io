---
title: 'Quickly Add or Delete an Infoblox host entry.'
date: '2015-01-29T00:33:20+00:00'
author: deaves
layout: post
permalink: /2015/01/quickly-add-or-delete-an-infoblox-host-entry/
categories:
    - Linux
    - 'Linux Admin'
    - 'Linux Scripts'
tags:
    - infoblox
    - linux
    - script
---

I was going through the published Infoblox /wapidoc documentation and decided to write a shell script that gives me the ability to bulk add/delete Infoblox HOST records at my office. This script uses [*cURL*](http://curl.haxx.se/) to post against an Infoblox grid master. You will need to make sure your user account has API permissions and the DNS zone association for the IPAM address range is configured to allow the particular domains host entries.

The following is the output of the script adding and deleting HOST records on an Infoblox in my lab.

## Adding/Updating a HOST record

```
$ ibHOST.sh -u -h testing -i 172.16.1.254
D: "record:host/ZG5zLmhvc3QkLjEuY29tLnNweC5nbG9iYWwuZGUtdGVzdGluZw:testing.domain.contoso.com/Internal%20View"
U: "record:host/ZG5zLmhvc3QkLjEuY29tLnNweC5nbG9iYWwuZGUtdGVzdGluZw:testing.domain.contoso.com/Internal%20View"

$ host testing
testing.domain.contoso.com has address 172.16.1.254
```

## Deleting a HOST record

```
$ ibHOST.sh -d -h testing
D: "record:host/ZG5zLmhvc3QkLjEuY29tLnNweC5nbG9iYWwuZGUtdGVzdGluZw:testing.domain.contoso.com/Internal%20View"

$ host testing
Host testing.domain.contoso.com not found: 3(NXDOMAIN)
```

This script has been very handy for most of my data center migrations. Using simple loop iteration to go through a list you can bulk add host records using the Infoblox WebAPI’s. This is not the best script to show off Infoblox WebAPI’s, but it gets the job done. If your looking to use this script, be very careful and test it before any mass runs! I take no responsibility if you damage anything in your environment.

## Script

Just an FYI; this script is a modification of a previous post I did in 2012 that uses nsupdate to update A records on a bind server: [Quickly update or delete a zonefile host entry.](/2012/01/quickly-update-or-delete-a-zonefile-host-entry/)

```bash
#!/bin/bash
## Created By: deaves
# Quickly Add or Delete an Infoblox host entry.
#
## Requires: curl, WebAPI enabled on Infoblox.
 
DOMAIN="domain.contoso.com"
SERVER="infoblox.contoso.com"
DNSVIEW="Internal View"
USER="joeuser:joepass"
 
##### Begin Script #####
 
function DELETE () {
### DELETE HOST ###
 echo -n "D: "
 curl -k -u ${USER} -X DELETE https://${SERVER}/wapi/v1.0/`curl -k -u ${USER} -X GET https://${SERVER}/wapi/v1.0/record:host -d name=${HOST}.${DOMAIN} 2> /dev/null | grep "_ref" | head -n1 | awk -F\" '{print $4}'` 2> /dev/null
 echo
}
 
function ADD () {
### Update DNS record for HOST ###
 echo -n "U: "
 curl -k -u ${USER} -H "Content-Type: application/json" -X POST https://${SERVER}/wapi/v1.0/record:host -d "{ \"ipv4addrs\":[{\"configure_for_dhcp\": false,\"ipv4addr\": \"${IPv4}\"}],\"name\": \"${HOST}.${DOMAIN}\",\"view\": \"${DNSVIEW}\"}" 2> /dev/null
 echo
}
 
function usage () {
  ### Display the script arguments.
  printf "Usage: $0 [-du] -h  -i \n\n"
  printf "Requires one option!\n"
  printf "\t-d: Delete a \"${DOMAIN}\" HOST record\n"
  printf "\t-u: Update/Add a \"${DOMAIN}\" HOST record\n\n"
}
 
while getopts "duh:i:" ARG; do
  case "${ARG}" in
    d) [ -z $ACTION ] && { ACTION="D"; };;
    u) [ -z $ACTION ] && { ACTION="U"; };;
    h) HOST="$(echo $OPTARG | tr [:upper:] [:lower:])";;
    i) IPv4="$OPTARG";;
    ?) echo "Invalid option -$OPTARG"; exit 1;;
  esac
done 2> /dev/null
 
if [ "$ACTION" == "U" ]; then
  [ -z "$IPv4" ] && { echo "Error: Missing IP" && exit 1; }
  [ "$(host ${HOST}.${DOMAIN} | awk '{print $NF}')" != "3(NXDOMAIN)" ] && { DELETE ;}
  ADD
elif [ "$ACTION" == "D" ]; then
  [ -z "$HOST" ] && { echo "Error: Missing HOST" && exit 1; }
  [ "$(host ${HOST}.${DOMAIN} | awk '{print $NF}')" != "3(NXDOMAIN)" ] && { DELETE ;}
else
  usage && exit 1;
fi
```
