---
title: 'Cisco Firepower Management Center (FMC) bulk import & delete objects'
date: '2018-08-08T16:22:02+00:00'
author: deaves
layout: post
permalink: /2018/08/cisco-firepower-management-center-fmc-bulk-import-delete-objects/
categories:
    - Cisco
    - Firewall
    - Linux
    - 'Linux Scripts'
    - Networking
tags:
    - cisco
    - curl
    - firepower
    - fmc
    - ftd
    - json
    - yaml
---

I’ve been doing a lot of migration work with the Cisco Firepower. I quickly noticed a general lack of tools to assist with migrations short of a few scattered and limited *(hopefully incomplete and will continue to be developed)* migration tools provided by Cisco:

- [enableMigrationTool.pl](https://www.cisco.com/c/en/us/td/docs/security/firepower/621/asa2ftd-migration/asa2ftd-migration-guide-621/asa2ftd_migration_procedure.html#id_35306): Run as root on sacrificial FMC / will only convert ACL &amp; NAT policies from an ASA config.
- [Cisco Firepower Migration Tool](https://www.cisco.com/c/en/us/products/security/firewalls/firepower-migration-tool.html): Runs under Windows and assists with migrating only ACL &amp; NAT policies from an ASA config.

If you are looking for tools to perform bulk rule changes or help convert from Layer4 rules to Layer7, like the PaloAlto [Migration tool](https://live.paloaltonetworks.com/t5/Expedition-Articles/What-is-Expedition/ta-p/215236), you are out of luck. That being said, as an engineer trying to use the FMC, I quickly found the experience of working within the firepower interface slow, tedious and generally painful. This has forced me to try to interface with the API’s as much as possible to save time and to avoid using the interface.

First thing; API access needs to be enabled on the FMC; by default they are, but if disabled you can enable them by going to **System > REST API Preferences** and enabling them. If API’s are enabled the documentation will be accessible via: **https://FMCHOST/api/api-explorer**

API’s on the FMC take an imperative approach and are simple to trigger and depending on the HTTP method you can command the FMC to make changes. There are 4 basic methods the FMC will accept:

- GET – Retrieves data from the specified object. GET is a read-only operation.
- PUT – Adds supplied information to the specified object; returns a 404 Resource Not Found error if the object does not exist.
- POST – Creates the object with the supplied information. POST operations are be followed with a payload consisting of JSON.
- DELETE – Delete and object.

Concerning the types of objects that can be modified; the following are currently supported:

- icmpv4objects
- icmpv6objects
- interfacegroups
- networkgroups
- networks
- portobjectgroups
- protocolportobjects
- ranges
- securityzones
- slamonitors
- urlgroups
- urls
- vlangrouptags
- vlantags

As far as I can tell vpn policies, flexconfig and other objects of the sort must be created by hand. The following json & script are very rudimentary, however it is a working example that uses cURL to perform a bulk import of objects into a firepower. At the very least the curl commands can be used as a reference in your own projects.

### fmc-objects.json: ASCII text, with very long lines

```json
[
  {
    "value": "10.0.0.0",
    "overridable": false,
    "name": "10.0.0.0_32",
    "type": "Hosts",
    "description": "Test REST API Object"
  },
  {
    "value": "10.0.0.1",
    "overridable": false,
    "name": "10.0.0.1_32",
    "type": "Hosts",
    "description": "Test REST API Object"
  },
  {
    "value": "10.0.0.2",
    "overridable": false,
    "name": "10.0.0.2_32",
    "type": "Hosts",
    "description": "Test REST API Object"
  },
  {
    "value": "10.0.0.3",
    "overridable": false,
    "name": "10.0.0.3_32",
    "type": "Hosts",
    "description": "Test REST API Object"
  },
  {
    "value": "10.0.0.4",
    "overridable": false,
    "name": "10.0.0.4_32",
    "type": "Hosts",
    "description": "Test REST API Object"
  },
  {
    "value": "10.0.0.5",
    "overridable": false,
    "name": "10.0.0.5_32",
    "type": "Hosts",
    "description": "Test REST API Object"
  },
  {
    "value": "10.0.0.6",
    "overridable": false,
    "name": "10.0.0.6_32",
    "type": "Hosts",
    "description": "Test REST API Object"
  },
  {
    "value": "10.0.0.7",
    "overridable": false,
    "name": "10.0.0.7_32",
    "type": "Hosts",
    "description": "Test REST API Object"
  },
  {
    "value": "10.0.0.8/29",
    "overridable": false,
    "name": "10.0.0.8_29",
    "type": "Networks",
    "description": "Test REST API Object"
  }
]
```

Because I do a lot with Ansible I have a strong preference to work with the YAML data serialization format. The script includes 2 unused functions that converts JSON to YAML and vice versa. I left those functions in this example script because they are handy and provide me flexibility of what and how I import. When posting to the FMC, its important to make sure the data is in JSON format and any metadata & link value trees have been popped. One thing I am lacking is a function to verify the object has been created, its easy to do by simply performing a GET after the POST, it just makes things slower and is simply not in this example.

### fmc-objects.sh: Bourne-Again shell script, ASCII text executable

```bash
#!/bin/bash
## Example script that can bulk import and delete objects from a Cisco FMC.
## Requires: python:PyYAML,shyaml
## 2018 (v.01) - Script from www.davideaves.com

firepower="192.0.2.10"
username="apiuser"
password="api1234"

# Convert JSON to YAML
j2y() {
 python -c 'import sys, yaml, json; yaml.safe_dump(json.load(sys.stdin), sys.stdout, default_flow_style=False)' 2> /dev/null
}

# Convert YAML to JSON
y2j() {
 python -c 'import sys, yaml, json; y=yaml.load(sys.stdin.read()); print json.dumps(y)' 2> /dev/null
}

fmcauth() {
 ### Post credentials and eval header return data.

 if [[ -z "${auth_epoch}" || "${auth_epoch}" -lt "$(($(date +%s) - 1500))" ]]
  then

   if [ -z "${X_auth_access_toke}" ]
    then eval "auth_epoch=$(date +%s)"
        eval "$(curl -skX POST https://${firepower}/api/fmc_platform/v1/auth/generatetoken \
          -H "Authorization: Basic $(printf "${username}:${password}" | base64)" -D - |\
          awk '/(auth|DOMAIN|global)/{gsub(/[\r|:]/,""); gsub(/-/,"_",$1); print $1"=\""$2"\""}')"
    else eval "auth_epoch=$(date +%s)"
         eval "$(curl -skX POST https://${firepower}/api/fmc_platform/v1/auth/refreshtoken \
          -H "X-auth-access-token: ${X_auth_access_token}" \
          -H "X-auth-refresh-token: ${X_auth_refresh_token}" -D - |\
          awk '/(auth|DOMAIN|global)/{gsub(/[\r|:]/,""); gsub(/-/,"_",$1); print $1"=\""$2"\""}')"
   fi

 fi
}


### Bulk add diffrent type objects into FMC

# Cat single line example containing diffrent objects & use awk to extract json stanza.
cat "fmc-objects.json" | awk -F'[}{]' '{ for (i=1; i 5) print "{"$i"}" }' |\
while read LINE
 do fmcauth

   # Determine Type & Value.
   TYPE="$(echo ${LINE} | shyaml get-value type | awk '{print tolower($0)}')"
   VALUE="$(echo ${LINE} | shyaml get-value value | awk '{print tolower($0)}')"

   echo -ne "$(date) - POST:${TYPE}>${VALUE} - MSG: "

   # Post new object to FMC.
   curl -skX POST https://${firepower}/api/fmc_config/v1/domain/${DOMAIN_UUID}/object/${TYPE} \
     -H "Content-Type: application/json" \
     -H "X-auth-access-token: ${X_auth_access_token}" \
     -d "${LINE}" && echo
done

### Bulk delete all unused objects from FMC...

# Valid objects types are: hosts icmpv4objects icmpv6objects interfacegroups networkgroups networks portobjectgroups protocolportobjects ranges securityzones slamonitors urlgroups urls vlangrouptags vlantags
for TYPE in hosts networks
 do fmcauth
    echo "### DELETEING ${TYPE} OBJECTS ###"

    # Get a list of objects.
    curl -skX GET https://${firepower}/api/fmc_config/v1/domain/${DOMAIN_UUID}/object/${TYPE}?limit=10000 \
     -H "X-auth-access-token: ${X_auth_access_token}" | j2y | shyaml get-value items 2> /dev/null | awk '/self:/{print $NF}' | while read SELF
    do  fmcauth
        echo -ne "$(date) - DELETE:${TYPE} - MSG: "

        # Delete the object.
        curl -skX DELETE ${SELF} -H "X-auth-access-token: ${X_auth_access_token}" && echo
    done
done
```
