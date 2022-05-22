---
title: 'HWIC-8A Base Config'
date: '2014-12-03T00:46:46+00:00'
author: deaves
layout: post
permalink: /2014/12/hwic-8a-base-config/
categories:
    - Cisco
    - Networking
tags:
    - cisco
    - config
    - HWIC-8A
---

Normally they are to expensive for what they do, but the other day I found a HWIC-8A from ebay at a good price. As a result, I now have remote Serial &amp; JTAG access to a bunch of test equipment via my Cisco Router. The following is a quick sample config I tossed together on how to configure it.

If needed the following is the pin-out to the Cisco Octal Cable: http://www.cisco.com/c/en/us/support/docs/dial-access/asynchronous-connections/14958-24.html

```tsx
! Create a AAA authentication policy that will
! not make the user supply local credentials to
! connect to the Async TTY's. 

aaa new-model
aaa authentication login TERMSERV none

! Create an ACL to control who can connect.
! Warning: Anyone will be able to connect to the
! tty's when transport is configured.

ip access-list standard TERMSERV
 remark *** TERMSERV ACCESS ***
 permit 10.0.0.0 0.255.255.255
 permit 172.16.0.0 0.15.255.255
 permit 192.168.0.0 0.0.255.255

! Need to change the physical-layer to async
! Interface descriptions correspond to the
! CAB-HD8-ASYNC cable each port will represent.

interface Serial0/0/0
 physical-layer async
 description [0-3/0]
!
interface Serial0/0/1
 physical-layer async
 description [0-3/2]
!
interface Serial0/0/2
 physical-layer async
 description [0-3/4]
!
interface Serial0/0/3
 physical-layer async
 description [0-3/6]
!
interface Serial0/0/4
 physical-layer async
 description [4-7/0]
!
interface Serial0/0/5
 physical-layer async
 description [4-7/2]
!
interface Serial0/0/6
 physical-layer async
 description [4-7/4]
!
interface Serial0/0/7
 physical-layer async
 description [4-7/6]

! Set transport type and bind ACL/AAA to the Async lines.

line 0/0/0 0/0/7
 access-class TERMSERV in vrf-also
 login authentication TERMSERV
 transport input all
 transport output all
```
