---
title: 'Create screen session to all ttyUSB devices'
date: '2015-01-09T22:55:33+00:00'
author: deaves
layout: post
permalink: /2015/01/create-screen-session-to-all-ttyusb-devices/
categories:
    - Linux
    - 'Linux Admin'
---

The following one-liner will create a new window for each /dev/ttyUSB port connected to the system. Assuming you’re in the dialout group. :)

```bash
for g in `groups`
 do [ "$g" == "dialout" ] &&
        {
          for TTY in /dev/ttyUSB*
           do TERM=`basename $TTY`
                screen -t "$TERM" $TTY 9600,-ixoff,-ixon || screen -s "$TERM"
           done
        }
done
```
