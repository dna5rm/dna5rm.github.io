---
title: 'Search for object matches in an ASA config.'
date: '2018-12-28T17:58:58+00:00'
author: deaves
excerpt: 'The following AWK script will munge an ASA config searching for any specified address or object name and will output any objects that reference it.'
layout: post
permalink: /2018/12/search-for-object-matches-in-an-asa-config/
categories:
    - AWK
    - Firewall
    - Linux
    - 'Linux Scripts'
    - Networking
tags:
    - asa
    - awk
    - cisco
    - firewall
    - migration
    - object-group
    - script
---

Having to parse ASA configs for migration purposes provides a never-ending source of reasons to write scripts. The following AWK script will munge an ASA config searching for any specified address or object name and will output any objects that reference it. This script is something I use in conjunction with the [ASA\_acls.sh](/2018/11/convert-asa-access-list-rules-to-a-parseable-yaml-format/) script to find security rules relating to an address. As far as I know this is the closest offline tool simmilar to the “Where Used” feature in ASDM for finding addresses.

## ASA\_obj.awk: awk script, ASCII text executable

```awk
#!/usr/bin/awk -f
## Search for object matches in an ASA config.
## 2018 (v.01) - Script from www.davideaves.com

### BEGIN ###

BEGIN {
  dig_range="y"
  dig_subnet="n"

  # Script arguments: ASA configuration + Search objects
  if ( ARGV[1] == "" ) {
    print "ERROR: No Input ASA config provided!" > "/dev/stderr"
    exit 1
  } else if ( ARGV[2] == "" ) {
    print "ERROR: No address or object to search for!" > "/dev/stderr"
    exit 1
  } else {
    # Saving everything after ARGV[1] in search_array.
    for (i = 2; i < ARGC; i++) {
      search_array[ARGV[i]] = ARGV[i]
      delete ARGV[i]
  } }
}

### FUNCTIONS ###

# Convert IP to Interger.
function ip_to_int(input) {
  split(input, oc, ".")
  ip_int=(oc[1]*(256^3))+(oc[2]*(256^2))+(oc[3]*(256))+(oc[4])
  return ip_int
}

# test if a string is an ipv4 address
function is_v4(address) {
  split(address, octet, ".")
  if ( octet[1] <= 255 && octet[2] <= 255 && octet[3] <= 255 && octet[4] <= 255 )
  return address
}
 
# convert number to bits
function bits(N){
  c = 0
  for(i=0; i<8; ++i) if( and(2**i, N) ) ++c
  return c
}

# convert ipv4 to prefix
function to_prefix(mask) {
  split(mask, octet, ".")
  return bits(octet[1]) + bits(octet[2]) + bits(octet[3]) + bits(octet[4])
}

### SCRIPT ###

//{ gsub(/\r/, "") # Strip CTRL+M

  ### LINE IS NAME ###
  if ( $1 ~ /^name$/ ) {

    name=$3; host=$2; type=$1
    for(col = 5; col <= NF; col++) { previous=previous" "$col }
    description=substr(previous,2)
    previous=""

    # Add to search_array
    for (search in search_array) if ( host == search ) search_array[name]
  }

  ### LINE IS OBJECT ### 
  else if ( $1 ~ /^object/ ) {

    tab="Y"
    name=$3
    type=$2
    if ( type == "service" ) service=$4
    previous=""

  } else if ( tab == "Y" && substr($0,1,1) == " " ) {

    # object is single host.
    if ( $1 == "host" ) {
      host=$NF
      for (search in search_array) if ( host == search ) search_array[name]
    }

    # object is a subnet
    else if ( $1 == "subnet" && dig_subnet == "y" ) {
      for (search in search_array) if ( is_v4(search) ) {

        NETWORK=ip_to_int($2)
        PREFIX=to_prefix($3)
        BROADCAST=(NETWORK + (2 ^ (32 - PREFIX) - 1))

        if ( ip_to_int(search) >= int(NETWORK) && ip_to_int(search) <= int(BROADCAST) ) {
          search_array[name]
      } }
    }

    # object is a range
    else if ( $1 == "range" && dig_range == "y" ) {
      for (search in search_array) if ( is_v4(search) ) {
        if ( ip_to_int(search) >= ip_to_int($2) && ip_to_int(search) <= ip_to_int($3) ) {
          search_array[name]
      } }
    }

    # object is group of other objects
    else if ( $2 ~ /(host|object)/ ) {
      for (search in search_array) if ( $NF == search ) search_array[name]
    }

    # object contains nat statement
    else if ( $1 == "nat" ) {
      for (search in search_array) if ( $NF == search ) search_array[name]
    }

    ### Debug everything else within an object
    #else { print "DEBUG:",$0 }

  }
  else { tab="" }

}

### END ###

END{
  if ( isarray(search_array) ) {
    print "asa_objects:"
    for (search in search_array) print "  -",search
  }
}
```
