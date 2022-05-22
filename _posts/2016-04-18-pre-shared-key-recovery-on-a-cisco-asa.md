---
title: 'Pre-shared Key Recovery on a Cisco ASA'
date: '2016-04-18T22:49:29+00:00'
author: deaves
layout: post
permalink: /2016/04/pre-shared-key-recovery-on-a-cisco-asa/
categories:
    - Cisco
    - Firewall
    - Networking
tags:
    - asa
    - cisco
    - firewall
    - psk
    - recovery
---

This quickie post is mainly for my own future benefit... The following is how you perform a pre-shared key recovery on a Cisco ASA. When you configure a PSK on a Cisco ASA and then review the configuration by doing a “*show running-config*“, all the passwords will be displayed as a bunch of \*\*\*’s from then on. There is a publicized, but not well know, way to view the full running-config by doing a “***more system:running-config***” which will allow you to view the running-config in its entirety. This command is nothing new and has apparently has been around since the PIX days.

Ref: [http://www.cisco.com/c/en/us/td/docs/security/asa/asa91/configuration/general/asa\_91\_general\_config/ref\_cli.html#52156](http://www.cisco.com/c/en/us/td/docs/security/asa/asa91/configuration/general/asa_91_general_config/ref_cli.html#52156)
