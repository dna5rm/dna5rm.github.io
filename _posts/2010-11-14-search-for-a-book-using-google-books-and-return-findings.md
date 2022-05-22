---
title: 'Search for a book using google books and return findings.'
date: '2010-11-14T16:39:29+00:00'
author: deaves
layout: post
permalink: /2010/11/search-for-a-book-using-google-books-and-return-findings/
categories:
    - 'Linux Scripts'
tags:
    - ebook
    - google
    - linux
---

```bash
#!/bin/bash
## Created by: deaves
# Dig a book by ISBN (or whatever).
#
## Requires: wget

QUERY="$1"
dbPATH="$(dirname "$0")/$(basename "$0" | awk -F\. '{print $1".db"}')/"

### Script Usage ###
function usage () {
  printf "ISBN dig\n
     Lookup a book by ISBN...\n\n"
  printf "Usage: $0 <isbn>\n"
}

[ -z "${1}" ] && { usage && exit 1; }

# Create DB directory to store query reqults
if [ ! -d "${dbPATH}" ]; then mkdir -p "${dbPATH}"; fi

### Dig using Google Books ###
if [ ! -e "${dbPATH}/G_${QUERY}" ]; then

  URL="http://books.google.com/books/feeds/volumes?q='${QUERY}'&start-index=1&max-results=1"

  wget "$URL" --quiet --output-document=- | sed 's/*//g;s/<dc></dc>/:/g' | grep ^: |\
   while IFS=: read X A B; do

    [ -z "${pA}" ] && { pA="0"; }
    if [ "${A}" == "${pA}" ] && [ "${A}" == "title" ]; then
      printf ": $(echo ${B} | sed 's/\ $//')"
    elif [ "${A}" == "${pA}" ]; then
      printf ", $(echo ${B} | sed 's/\ $//')"
    elif [ "${A}" == "identifier" ]; then
      printf "\n$(echo ${A} | tr [:lower:] [:upper:])=\""
    else
      printf "\n$(echo ${A} | tr [:lower:] [:upper:])=\"$(echo ${B} | tr -d [:cntrl:] | sed 's/\ $//')"
    fi

    # Remember vars to compare against the next pass.
    pA="${A}"; pB="${B}"; pC="${C}"

  done | sed 's/$/\"/;s/",\ /"/;s/\&amp\;/\&/g;s/\&quot\;/'\''/g' | grep -v ^"\"" > "${dbPATH}/G_${QUERY}"
fi

### Display the results ###
if [ -s "${dbPATH}/G_${QUERY}" ]; then
  cat "${dbPATH}/G_${QUERY}"
else
  rm "${dbPATH}/G_${QUERY}"
  echo "No results."
fi</isbn>
```
