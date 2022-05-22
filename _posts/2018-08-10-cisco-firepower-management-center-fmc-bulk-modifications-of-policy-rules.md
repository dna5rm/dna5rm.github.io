---
title: 'Cisco Firepower Management Center (FMC) bulk modifications of policy rules.'
date: '2018-08-10T08:05:24+00:00'
author: deaves
layout: post
permalink: /2018/08/cisco-firepower-management-center-fmc-bulk-modifications-of-policy-rules/
categories:
    - Cisco
    - Firewall
    - Linux
    - 'Linux Scripts'
    - Networking
tags:
    - api
    - cisco
    - curl
    - firepower
    - fmc
    - ftd
    - policy
    - script
---

As stated in my previous [post](/2018/08/cisco-firepower-management-center-fmc-bulk-import-delete-objects/), the Cisco Migration tools are very limited and Cisco has not shared any tools publicly that can perform bulk modifications. When migrating to a firepower unless all the previous access rules on the ASA had logging enabled any rule converted will continue to not have logging enabled. This is a major bummer for auditors, security teams or for anyone looking to eventually migrate L4 rules to L7 rules.

Just because I can, the following example script will bulk enable *logBegin* & *sendEventsToFMC* on **all** policy rules on a Firepower (FMC) appliance. This script is intended to be a reference, however modifying policy rules can broadly be broken down into the following steps:

1. GET the accessrule
2. Purge the metadata & link value trees from the json
3. Change, add or remove whatever values you want
4. PUT the accessrule

```bash
#!/bin/bash
## Example script that can bulk modify policy rules on a Cisco FMC.
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

# Pop forbidden put values (metadata, links)
jpop() {
 python -c 'import sys, json; j=json.load(sys.stdin); j.pop("metadata"); j.pop("links"); print json.dumps(j)' 2> /dev/null
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

# Get a list of all access control policies.
fmcauth && curl -skX GET https://${firepower}/api/fmc_config/v1/domain/${DOMAIN_UUID}/policy/accesspolicies \
 -H "X-auth-access-token: ${X_auth_access_token}" | j2y | shyaml get-value items | awk '/self:/{print $NF}' | while read POLICY
do fmcauth

   # Get a list of all access rules in each policy.
   curl -skX GET ${POLICY}/accessrules?limit=10000 -H "X-auth-access-token: ${X_auth_access_token}" |\
   j2y | shyaml get-value items 2> /dev/null | awk '/self:/{print $NF}' | while read RULE
   do

      # ID of the access rule.
      UUID="$(basename ${RULE})"

      # Collect the access rule resource and pop forbidden values.
      RESOURCE=`curl -skX GET ${RULE} -H "X-auth-access-token: ${X_auth_access_token}" | jpop`

      # Modify only if resource exists.
      if [[ ${RESOURCE} != *"Resource not found."* ]]
       then echo -ne "$(date) - PUT:${UUID} - MSG: "

       # Use sed to modify values in payload before putting back to FMC.
       curl -skX PUT ${RULE} \
        -H "Content-Type: application/json" \
        -H "X-auth-access-token: ${X_auth_access_token}" \
        -d "$(echo ${RESOURCE} | j2y | sed 's/logBegin: false/logBegin: true/;s/sendEventsToFMC: false/sendEventsToFMC: true/' | y2j)" && echo
      fi

   done

done
```
