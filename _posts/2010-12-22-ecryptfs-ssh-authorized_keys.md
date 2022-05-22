---
title: 'ecryptfs & ssh authorized_keys'
date: '2010-12-22T07:26:13+00:00'
author: deaves
layout: post
permalink: /2010/12/ecryptfs-ssh-authorized_keys/
categories:
    - 'Linux Admin'
    - 'Linux Security'
tags:
    - ecryptfs
    - linux
    - ssh
---

```bash
ecryptfs-umount-private
chmod +w $HOME
mkdir -m 700 .ssh
echo "<your public key>" > $HOME/.ssh/authorized_keys
chmod 600 $HOME/.ssh/authorized_keys
chmod -w $HOME
ecryptfs-mount-private
```
