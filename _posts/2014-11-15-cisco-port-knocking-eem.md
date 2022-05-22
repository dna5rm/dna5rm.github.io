---
title: 'Dynamic Cisco Port Knocking EEM applet'
date: '2014-11-15T22:22:33+00:00'
author: deaves
layout: post
permalink: /2014/11/cisco-port-knocking-eem/
categories:
    - Cisco
    - EEM
    - 'Linux Security'
    - Networking
tags:
    - cisco
    - eem
    - 'port knocking'
---

There is nothing new about port knocking to hide remote access to a remote system or network. However its usually implemented as a hack thats done on a single host thats sitting at a remote site. If your running a Cisco router the only method to get port knocking working is to create an EEM applet. I have seen several port knocking EEM applets online, but none of them seem very good and they usually work by swapping out a less secure ACL for a more secure one.

After watching my server logs get obliterated by some knuckle head trying to brute force their way into it, I decided to get port knocking working on a Cisco router. The following EEM applet uses an extended ACL thats tied to the inbound WAN interface of the router. The ACL has a permit statement that will log a certain packet type; the logs will be updated with a notification that contains the originating IP address. The EEM applet monitors the logs and will be triggered when it sees the ACL. The applet will then pull out the knocking IP address and temporarily add it to the same inbound ACL allowing enough time to establish a connection from the originating machine. After 15 seconds have passed, the script will drop the permit line that was added to the ACL.

```tsx
!! SAMPLE ACL !!

ip access-list extended outside-in4
 remark *** KNOCK ***
 permit udp any any eq 65535 log
 remark *** TRUSTED ***
 permit tcp any any established
 remark *** DENIED ***
 deny   tcp any any
 remark *** PERMITED ***
 permit ip any any

!! WAN Interface !!

interface Cable-Modem0/1/0
 ip access-group outside-in4 in

!! KNOCK_ACL env Variable !!

event manager environment KNOCK_ACL outside-in4

!! Port Knocking EEM applet !!

event manager applet KNOCK
 event syslog pattern "%SEC-6-IPACCESSLOGP: list $KNOCK_ACL permitted *"
 action 1.0 regexp "[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+" $_syslog_msg ADDR
 action 1.1 regexp "\([0-9]+\)," "$_syslog_msg" PORT
 action 1.2 regexp "[0-9]+" "$PORT" PORT 
 action 2.0 syslog msg "Received a knock from $ADDR on port $PORT..."
 action 2.1 syslog msg "Adding $ADDR to the $KNOCK_ACL ACL"
 action 3.0 cli command "enable"
 action 3.1 cli command "configure terminal"
 action 3.2 cli command "ip access-list extended $KNOCK_ACL"
 action 3.3 cli command "1 permit tcp host $ADDR any eq 22"
 action 4.0 wait 15
 action 5.0 syslog msg "Removing $ADDR to the $KNOCK_ACL ACL"
 action 6.0 cli command "no permit tcp host $ADDR any eq 22"
 action 6.1 cli command "exit"
```

The only tricky thing in the above example is creating a good regex expression… Knocking with a UDP/65535 packet and immediately connecting to the routers WAN IP address will allow you to SSH into a server on the other side. The following is me crafting a simple UDP packet using hping3 under Linux.

```bash
hping3 -2 ROUTERIP -p 65535 -c 1; ssh ROUTERIP
```

## A few caveats:

- Usually any/any ACL’s are not good, but in my case, this is a home router doing PAT and a DHCP client on the WAN interface.
- Show active EEM policies: ***show event manager policy active***
- Show EEM history: ***show event manager history events***
- Validate the ACL is getting hit: ***show access-list outside-in4***
- The default EEM watchdog will terminate the applet after 20 seconds. *MAXRUN* will need to be changed if you want the applet to wait longer then 15 seconds before auto terminating.
- The ACL can be modified to log packet options such as special ToS, DSCP values in addition to ports.
- I recommend not using other log statements in the same ACL, doing so will require making a more custom applet.
- If your trying to log into the router itself via an ACL on a TTY line be mindful of any service-polices you have bound to the control-plane.
