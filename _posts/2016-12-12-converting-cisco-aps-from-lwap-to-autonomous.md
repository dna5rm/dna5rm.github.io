---
title: 'Convert Cisco APs from LWAP to Autonomous'
date: '2016-12-12T22:13:26+00:00'
author: deaves
layout: post
permalink: /2016/12/converting-cisco-aps-from-lwap-to-autonomous/
categories:
    - Cisco
    - Networking
    - Wireless
tags:
    - AIR-AP1131AG-N-K9
    - autonomous
    - cisco
    - wireless
---

A lot of companies are pulling out their old Cisco wireless infrastructure to upgrade or replace it all together. As a result its pretty easy to get your hands on older Cisco AP’s, unfortunately by default they require a special controller in order to function. If you want to turn your old Cisco LWAP AP into something other than a paperweight, you either need to get an older controller or convert the AP to Autonomous. The following snippet is what I had to do to convert my Cisco AP from LWAP to Autonomous. Surprisingly they don’t cover this sort of thing on the CCNA wireless exam; or at least not back when I took it.

Before you begin, just make sure you get the proper “**k9w7**” autonomous code; bear in mind that “*k9w8*” is the lightweight code. In my case I had an old Aironet 1130 AG Access Point, so I had to get my hands on the *c1130-k9w7-tar.124-25d.JA.tar* code. I also needed to setup a TFTP server inside the same vlan as the AP. After I consoled into the AP and logged in to it I had to do the following...

```tsx
debug lwapp console cli
debug lwapp client no-reload

config t
int fa 0
 ip address 10.0.0.1 255.255.255.224
end

archive download-sw /force-reload /overwrite tftp://10.0.0.2/c1130-k9w7-tar.124-25d.JA.tar
```
