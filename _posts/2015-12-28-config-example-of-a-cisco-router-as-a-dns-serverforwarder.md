---
title: 'Config example of a Cisco router as a DNS server/forwarder.'
date: '2015-12-28T16:51:27+00:00'
author: deaves
layout: post
permalink: /2015/12/config-example-of-a-cisco-router-as-a-dns-serverforwarder/
categories:
    - Cisco
    - Networking
tags:
    - cisco
    - 'config example'
    - 'dns server'
    - 'ip dns server'
---

For a quick and dirty DNS server you can configure a Cisco router. In the following config snippet I have configured a router as a DNS forwarder. Any ip host statements entered in the router will be resolvable by the clients.

```tsx
!!! Host statements will be resolvable by clients !!!
ip host rtr.SITE.LAN     192.168.0.1
ip host gi0-0-0.SITE.WAN 10.0.0.254
ip host gi0-0-1.SITE.LAN 192.168.0.1
ip host servera.SITE.LAN 192.168.0.10
ip host serverb.SITE.LAN 192.168.0.11
ip host serverc.SITE.LAN 192.168.0.12
ip host serverd.SITE.LAN 192.168.0.13
ip host servere.SITE.LAN 192.168.0.14

!!! ACL to limit who can query the DNS server service !!!
ip access-list standard RFC1918-dns
 permit 10.0.0.0 0.255.255.255
 permit 172.16.0.0 0.15.255.255
 permit 192.168.0.0 0.0.255.255

!!! DNS name-list is used to control what zones/hosts can be queried !!!
ip dns name-list 1 permit .*

!!! Create a DNS view !!!
ip dns view default
 domain name-server 8.8.8.8
 domain name-server 8.8.4.4
 domain name SITE.LAN
 dns forwarding source-interface GigabitEthernet0/0/0

!!! Create a DNS view-list !!!
ip dns view-list LAN
 view default 1
  restrict source access-group RFC1918-dns
  restrict name-group 1

!!! Enable DNS server service and use the view-group !!!
ip dns server view-group LAN
ip dns server
```

If running an ISR g3 I recommend upgrading to at least 15.5(3)S1a.
