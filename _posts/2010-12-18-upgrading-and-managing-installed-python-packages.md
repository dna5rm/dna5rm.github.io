---
title: 'Upgrading and managing installed Python packages'
date: '2010-12-18T11:58:04+00:00'
author: deaves
layout: post
permalink: /2010/12/upgrading-and-managing-installed-python-packages/
categories:
    - 'Linux Admin'
tags:
    - linux
    - python
---

Make sure **python-setuptools** is installed on system.

Install Yolk to query installed packages: 

```bash
easy_install yolk
```

Check PyPI for updates on packages:

```bash
yolk -U
```

easy_install can be used to upgrade any installed package:

```bash
yolk -a | cut -d ‘ ‘ -f 1 | xargs easy_install -U
```
