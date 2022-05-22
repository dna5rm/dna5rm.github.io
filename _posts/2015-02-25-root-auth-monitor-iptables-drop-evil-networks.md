---
title: 'Root Auth Monitor: iptables DROP evil networks'
date: '2015-02-25T00:51:53+00:00'
author: deaves
layout: post
permalink: /2015/02/root-auth-monitor-iptables-drop-evil-networks/
categories:
    - Linux
    - 'Linux Admin'
    - 'Linux Scripts'
    - 'Linux Security'
    - Networking
tags:
    - auth
    - iptables
    - linux
    - monitor
    - root
    - security
---

The following is an [upstart](http://upstart.ubuntu.com/) script that monitors &amp; blocks networks that fail to log into your Ubuntu server as root. Its great script to stop brute force logins to your server.

The following are a couple commands for reference:

Start/Stop the script...
> start tty12

> stop tty12

List INPUT rules w/line numbers...
> iptables -L INPUT -n -line-numbers

Delete an INPUT rule by line number...
> iptables -D INPUT 1

```bash
# /etc/init/tty12 - Root Auth Monitor: iptables DROP evil networks
# Required modifying/adding PermitRootLogin & AllowUsers to /etc/ssh/sshd_config

start on runlevel [23] and not-container
stop on runlevel [!23]

respawn

script
exec > /dev/tty12
tail -fn0 /var/log/auth.log | while read LINE
do echo "$LINE" | grep ": Failed password for invalid user root from"
 if [ $? = 0 ]
  then
   whois -h whois.cymru.com " -p $(echo "$LINE" | awk '{print $13}')" | grep ^[0-9] | sed 's/ *| */|/g' |\
   while IFS="|" read AS IP PREFIX NAME
    do iptables -I INPUT -s $PREFIX -j DROP -m comment --comment "AS$AS: $NAME"
   done
 fi
done
end script
```
