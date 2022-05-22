---
title: 'Identify duplicate files in current working directory by file hash and take user specified action.'
date: '2014-08-14T15:58:43+00:00'
author: deaves
layout: post
permalink: /2014/08/identify-duplicate-files-in-current-working-directory-by-file-hash-and-take-user-specified-action/
categories:
    - 'Linux Admin'
    - 'Linux Scripts'
---

There are a lot of programs out there that will locate duplicate files on a filesystem. I however, prefer to use standard system utilities on my home system. So I wrote a quick shell script to identify duplicate files and either create hard links between them or prompt me on which files to delete outright.

Feel free to review, modify or use this script however you see fit. Remember you do so at your own risk!

```bash
#!/bin/bash
## Created by: deaves
# Identify duplicate files in current working directory by file hash and take user specified action.
#
## Requires: dialog, findutils

# Required script variables.
filehash_dump="/tmp/$USER-$$.hash"
dialog_select="/tmp/$USER-$$.select"
report_file="$HOME/uDupeReport-$$.txt"

# FUNCTION: End Script if error.
DIE() {
 echo "ERROR: Validate \"$_\" is installed and working on your system."
 exit 0
}

# Validate script requirements are meet.
type -p dialog > /dev/null || DIE

# Create the filehash_dump file containing MD5 finger prints.
find ./ -type f -exec md5sum $1 {} \; | tee "$filehash_dump"

# Prompt user for next action to take w/6 hour wait before continuing on.
echo
read -t 21600 -p "DELETE or LINK duplicate files?
If NO response create a report: $report_file
Type command in CAPS: " OPT
echo

# Begin loop with all file hashes.
awk '{print $1}' "$filehash_dump" | sort | uniq | while read HASH
 do
 COUNT=0

 # Read all the files of the hash into an Array.
 eval FILES=( "$(grep ^"$HASH" "$filehash_dump" | sed 's/'"$HASH"'  //g;s/^/"/g;s/$/"/g')" )

if [ "$OPT" == "DELETE" ]; then

 # If more than a single file exists then take action.
 if [ "${#FILES[@]}" -gt "1" ]; then

  # Prompt user on which file to keep and store it in dialog_select.
  dialog --backtitle "$0 - uDupe File Killer" \
    --title "Select the file to keep. All others files will be deleted!" \
    --menu "HASH: $HASH\nTYPE:$(file "${FILES[${COUNT}]}" | awk -F': ' '{$1=""; print}')" \
    17 70 14 \
  $(while [ "$COUNT" -lt "${#FILES[@]}" ]; do
   printf " $COUNT "
   printf "${FILES[${COUNT}]}" | sed 's/ /_/g'
   let COUNT++
  done) 2> "$dialog_select"

  # Trap error code from dialog & perform actions.
  if [ "$?" == "0" ]; then

   # User selected a file.
   ANS=`cat "$dialog_select"`; rm "$dialog_select"
   unset FILES[${ANS}]

   # Perform action against unselected files.
   for file in "${FILES[@]}"; do
    rm "$file"
   done

  else

   # User choose to cancel.
   rm "$dialog_select"
   rm "$filehash_dump"
   break

  fi

 fi

elif [ "$OPT" == "LINK" ]; then

 # hardLink all the duplicate files to save disk space.
 if [ "${#FILES[@]}" -gt "1" ]; then
  echo "Creating hardlink: $HASH"
  echo " > ${FILES[0]}"
  ORIG="${FILES[0]}"; unset FILES[0]
  for file in "${FILES[@]}"; do
   echo " > $file"
   rm "$file"
   ln "$ORIG" "$file"
  done
 fi

else

 # Just Report all the duplicate files.
 [ ! -f "$report_file" ] && { printf "uDupe Report: $(pwd)\n$(date)\n\n" > "$report_file" ;}
 if [ "${#FILES[@]}" -gt "1" ]; then
  echo "HASH: $HASH"
  echo "TYPE:$(file "${FILES[${COUNT}]}" | awk -F': ' '{$1=""; print}')"
  for file in "${FILES[@]}"; do
   echo " > $file"
  done
  echo
 fi | tee -a "$report_file"

fi

done

# Script is finished, delete the filehash_dump file.
[ -e "$filehash_dump" ] && { rm "$filehash_dump" ; }
```
