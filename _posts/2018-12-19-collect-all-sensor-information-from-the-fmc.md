---
title: 'Collect all sensor information from the FMC.'
date: '2018-12-19T18:53:54+00:00'
author: deaves
excerpt: 'The following is a quick script that will collect all sensor information from a Firepower Management Center using the devicerecords API and save that information to a CSV file.'
layout: post
permalink: /2018/12/collect-all-sensor-information-from-the-fmc/
categories:
    - Cisco
    - Firewall
    - 'Linux Scripts'
    - Networking
    - Uncategorized
tags:
    - api
    - cisco
    - curl
    - firepower
    - fmc
    - script
    - sourcefire
---

Eventually I plan on refactoring all my firepower scripts into Ansible Playbooks. But in the meanwhile the following is a quick script that will collect all sensor information from a Firepower Management Center and save that information to a CSV file. The output is pretty handy for migrations and general data collection.

```bash
#!/bin/bash
## Collect all sensor devicerecords from a FMC.
## Requires: python:PyYAML,shyaml
## 2018 (v.01) - Script from www.davideaves.com

username="fmcusername"
password="fmcpassword"

FMC="192.0.2.13 192.0.2.14 192.0.2.15 192.0.2.16 192.0.2.17 192.0.2.18 192.0.2.21 192.0.2.22 192.0.2.23"

### Convert JSON to YAML.
j2y() {
 python -c 'import sys, yaml, json; yaml.safe_dump(json.load(sys.stdin), sys.stdout, default_flow_style=False)' 2> /dev/null
}

### Convert YAML to JSON.
y2j() {
 python -c 'import sys, yaml, json; y=yaml.load(sys.stdin.read()); print json.dumps(y)' 2> /dev/null
}

echo "FMC,healthStatus,hostName,model,name," > "$(basename ${0%.*}).csv"

# Itterate through all FMC devices
for firepower in ${FMC}
 do eval "$(curl -skX POST https://${firepower}/api/fmc_platform/v1/auth/generatetoken \
        -H "Authorization: Basic $(printf "${username}:${password}" | base64)" -D - |\
        awk '/(auth|DOMAIN|global)/{gsub(/[\r|:]/,""); gsub(/-/,"_",$1); print $1"=\""$2"\""}')"

    ### Get expanded of list devices
    curl -skX GET "https://${firepower}/api/fmc_config/v1/domain/${DOMAIN_UUID}/devices/devicerecords?offset=0&limit=1000&expanded=true" -H "X-auth-access-token: ${X_auth_access_token}" |\
     j2y | awk 'BEGIN{ X=0; }/^(-|  [a-z])/{if($1 == "-") {X+=1; printf "'''${firepower}''',"} else if($1 == "healthStatus:" || $1 == "hostName:" || $1 == "model:" || $1 == "name:") {printf $NF","} else if($1 == "type:") {printf "\n"}}'

done >> "$(basename ${0%.*}).csv"
```
