---
title: 'Collect and archive all runtime information, statistics and status on F5 systems'
date: '2018-02-04T00:24:38+00:00'
author: deaves
layout: post
permalink: /2018/02/collect-runtime-information-on-f5-systems/
categories:
    - F5
    - Linux
    - 'Linux Scripts'
    - 'Load Balancing'
    - Networking
tags:
    - archive
    - awk
    - backup
    - base64
    - collect
    - f5
    - f5info
    - f5info.sh
    - runtime
    - script
    - statistics
    - status
---

Last march I posted a TCL/Expect script ([rtrinfo.exp](/2017/03/expect-backup-cisco-device-configs/)) to backup configs and regularly collect runtime information via show commands on Cisco devices for archival purposes. It’s proven itself to be very useful and replaces the need to purchase convoluted commercial software to archive device configs. Not only do I use it, but I know of a few companies that have adopted it as well. Recently I needed something similar that could collect and archive all runtime information, statistics and status on F5 systems; the following is that script.

The script works by pushing a small, base64 encoded, command string to the F5 to be executed. The command string simply does a "*tmsh -q show \\?*" to get a list of all show commands based on the enabled modules. The available runtime information is collected and piped to a for loop that runs all available show commands.

```bash
echo "Zm9yIE1PRCBpbiBgdG1zaCAtcSBzaG93IFw/IHwgc2VkIC1uIC1lICcvTW9kdWxlczovLC9PcHRpb25zOi9wJyB8IGF3ayAnL14gIC97cHJpbnQgJDF9J2A7IGRvIHRtc2ggLXEgc2hvdyAkTU9EIDI+IC9kZXYvbnVsbDsgZG9uZQ==" | base64 -d
for MOD in `tmsh -q show \? | sed -n -e '/Modules:/,/Options:/p' | awk '/^  /{print $1}'`; do tmsh -q show $MOD 2> /dev/null; done
```

All the output from the F5 is collected and some awk-foo is used to determine appropriate output destination on a per-line basis. The while loop appends the line to the appropriate file. Additionally all previous output archived to an appropriately named tar.gz file. I have also added the ability to silence the output, specify an output path override and to use root’s ssh private key instead of using the password (for running via cron).

## f5info.sh: Bourne-Again shell script, ASCII text executable

```bash
#!/bin/bash
## Collect and archive all runtime information, statistics and status on F5 systems.
## 2018 (v1.0) - Script from www.davideaves.com
 
OUTDIR="."
 
### Script Functions ###
function USAGE () {
 # Display the script arguments.
 printf "Usage: $0 -d bigip -i id_rsa -p path\n\n"
 printf "Requires:\n"
 printf "\t-d: Target F5 system.\n"
 printf "Options:\n"
 printf "\t-i: Private id_rsa of root user.\n"
 printf "\t-p: Destination of output directory.\n"
 printf "\t-q: Quiet, do not show anything.\n"
}
 
function CLEANUP {
 # Cleanup after the script finishes.
 [ -e "${IDENTITY}" ] && { rm -rf "${IDENTITY}"; }
}
 
### Get CLI options ###
while getopts "d:i:p:q" ARG; do
 case "${ARG}" in
  d) F5="${OPTARG^^}";;
  i) trap CLEANUP EXIT
     IDENTITY="$(mktemp)"
     chmod 600 "${IDENTITY}" && cat "${OPTARG}" > "${IDENTITY}";;
  p) OUTDIR="${OPTARG}";;
  q) QUIET="YES";;
 esac
done 2> /dev/null
 
### Display USAGE if F5 not defined ###
[ -z "${F5}" ] && { USAGE && exit 1; }
 
### Archive & Create OUTDIR ###
if [ -d "${OUTDIR}/${F5}" ]
 then ARCHIVE="${OUTDIR}/${F5}_$(date +%Y%m%d -d @$(stat -c %Y "${OUTDIR}/${F5}")).tar.gz"
 
  [ -e "${ARCHIVE}" ] && { rm -f "${ARCHIVE}"; }
  [ -z "${QUIET}" ] && { echo "Archiving: ${ARCHIVE}"; }
  tar zcfP "${ARCHIVE}" "${OUTDIR}/${F5}" && rm -rf "${OUTDIR}/${F5}"
 
fi && ssh -q -o StrictHostKeyChecking=no `[ -r "${IDENTITY}" ] && { echo -i "${IDENTITY}"; }` root@${F5} \
'bash -c "$(base64 -di <<< Zm9yIE1PRCBpbiBgdG1zaCAtcSBzaG93IFw/IHwgc2VkIC1uIC1lICcvTW9kdWxlczovLC9PcHRpb25zOi9wJyB8IGF3ayAnL14gIC97cHJpbnQgJDF9J2A7IGRvIHRtc2ggLXEgc2hvdyAkTU9EIDI+IC9kZXYvbnVsbDsgZG9uZQ==)"' |\
 awk 'BEGIN{
  FS=": "
 }
 // {
  gsub(/[ \t]+$/, "")
 
  # LN buffer
  LN[1]=LN[0]
  LN[0]=$0
 
  # Build OUTPUT variable
  ## LN ends with "{" - special case header
  if(substr(LN[0],length(LN[0]),1) == "{") {
   COUNT=split(LN[0],FN," ") - "1"
   for (i = 1; i <= COUNT; i++) FILE=FILE FN[i] "_"
   OUTPUT=substr(FILE, 1, length(FILE)-1)
  }
  ## LN does not contain "::" but is a header
  else if(OUTPUT == "" && LN[0] ~ /^[a-z,A-Z]/) {
   OUTPUT=LN[0]
  }
  ## LN contains "::" and is a header
  else if(LN[0] ~ /^[A-Z].*::[A-Z]/) {
   gsub(/::/,"-")
   DIR=gensub(/\ /, "_", "g", tolower($1))
   FILE=gensub(/[ ,:].*/, "", "g", $2)
   if(FILE != "") {
    OUTPUT=DIR"/"FILE
   } else {
    OUTPUT=DIR
   }
  }
 
  # Print OUTPUT & LN buffer
  if(OUTPUT != "") print(OUTPUT"<"LN[1])
 }
 END{
  print(OUTPUT"<"LN[0])
 }' | while IFS="<" read OUTPUT LN
  do
 
     if [ ! -w "${OUTDIR}/${F5}/${OUTPUT}" ]
      then [ -z "${QUIET}" ] && { echo "Saving: ${OUTDIR}/${F5}/${OUTPUT}"; }
           install -D /dev/null -m 644 "${OUTDIR}/${F5}/${OUTPUT}"
     fi && echo "${LN}" >> "${OUTDIR}/${F5}/${OUTPUT}"
 
 done
```
