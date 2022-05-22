---
title: 'Monitoring Cisco AP Dot11 Associations in Cacti'
date: '2016-01-14T21:07:04+00:00'
author: deaves
layout: post
permalink: /2016/01/cisco-ap-dot11-associations-in-cacti/
categories:
    - Cacti
    - Cisco
    - Networking
    - Wireless
tags:
    - ap
    - cacti
    - cisco
    - dot11
    - graph
    - template
    - wireless
---

This [Cacti](http://www.cacti.net) template should work with any autonomous Cisco AP. It will SNMP poll and display all active Cisco AP Dot11 Associations in Cacti. Note the AP I am testing with has an **AIR-RM3000AC-A-K9** module, giving me an extra radio.

![Cisco Dot11 - Active Wireless Clients](/assets/Cisco-Dot11-Active-Wireless-Clients.png)

Template: [*cacti_graph_template_cisco_dot11_-_active_wireless_clients.zip*](/assets/cacti_graph_template_cisco_dot11_-_active_wireless_clients.zip)

If you do not have a 802.11AC radio installed in your AP then after importing you may need to modify the Graph Template and remove all the Radio2 graph template items; not doing so may cause the graph not to display properly.

SNMP OIDs queried: [*\[SOURCE\]*](http://tools.cisco.com/Support/SNMP/do/BrowseOID.do?objectInput=cDot11ActiveWirelessClients&translate=Translate&submitValue=SUBMIT)

- **ActiveWirelessClients *(for 2.4Ghz radio)* = OID:** *.1.3.6.1.4.1.9.9.273.1.1.2.1.1.1*
- **ActiveWirelessClients *(for 5Ghz radio)* = OID:** *.1.3.6.1.4.1.9.9.273.1.1.2.1.1.2*
- **ActiveWirelessClients *(AIR-RM3000AC-A-K9)* = OID:** *.1.3.6.1.4.1.9.9.273.1.1.2.1.1.10*

**This Cacti template will import/update the following items:**

### GPRINT Preset

- Normal
- Exact Numbers

### Data Input Method

- Get SNMP Data

### Data Template

- Cisco Dot11 – Radio0 Associations
- Cisco Dot11 – Radio1 Associations
- Cisco Dot11 – Radio2 Associations

### Graph Template

- Cisco Dot11 – Active Wireless Clients
