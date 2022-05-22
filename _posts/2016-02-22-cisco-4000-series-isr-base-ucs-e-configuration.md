---
title: 'Cisco 4000 series ISR Base UCS-E Configuration'
date: '2016-02-22T17:39:43+00:00'
author: deaves
layout: post
permalink: /2016/02/cisco-4000-series-isr-base-ucs-e-configuration/
categories:
    - Cisco
    - Networking
tags:
    - cisco
    - ISR4331
    - ucs
    - UCS-E140S-M2
    - ucse
---

I have been replacing a lot of older Cisco ISR routers with 4000 series ISR’s lately. One of the more common things I have seen companies order with the new 4000 series routers are UCS-E blades; especially for smaller sites that don’t any servers. Unfortunately IOS-XE is still relatively new and it can be difficult to find proper configuration guides or working configs. As a result I have seen a lot of bad setups where engineers do not use the internal EVC link for UCS-E connectivity. Instead they cable the UCS-E external ports directly back into the router or cable it directly to the LAN switch. While this works, they are essentially running it as it was a separate device on the network and not part of the router. In this post I will provide a base UCS-E configuration to get people quickly up and running.

![UCS-E140S-M2](/assets/UCS-E140S-M2.png)

### Example IP allocation:

- /29 for CIMC &amp; ESXi Management.
> Example: 10.0.0.240/29
- /27 for UCS-E Server Vlan.
> Example: 10.0.0.128/27

When push comes to shove its best to view/treat the BDI interface, that’s tied to the ucse1/0/1 service instance, no different than a SVI on a L3 switch.

```tsx
ucse subslot 1/0
 imc access-port shared-lom console
 imc ip address 10.0.0.242 255.255.255.248 default-gateway 10.0.0.241
!
interface ucse1/0/0
 description *** UCS - Internal L3 Management (10.0.0.240/29) ***
 ip address 10.0.0.241 255.255.255.248
 negotiation auto
 switchport mode trunk
!
interface ucse1/0/1
 description *** UCS - Internal L2 Interface ***
 no ip address
 negotiation auto
 switchport mode trunk
 !
 service instance 1 ethernet
  description *** Server Vlan EVC ***
  encapsulation dot1q 1
  bridge-domain 1
 !
!
interface BDI1
 description Server Vlan (10.0.0.128/27)
 ip address 10.0.0.129 255.255.255.224
 no ip redirects
 no ip unreachables
 no ip proxy-arp
 encapsulation dot1Q 1
```

For further reading on EVC’s the following blog post is really good:
- [EVC: Flexible Service Mapping](http://ccie-in-3-months.blogspot.com/2009/09/evc-flexible-service-mapping.html)
