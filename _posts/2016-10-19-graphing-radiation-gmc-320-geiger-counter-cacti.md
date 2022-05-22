---
title: 'Graphing radiation with a GMC-320 geiger counter using Cacti'
date: '2016-10-19T19:35:34+00:00'
author: deaves
layout: post
permalink: /2016/10/graphing-radiation-gmc-320-geiger-counter-cacti/
categories:
    - Cacti
    - Linux
    - 'Linux Scripts'
---

A few years ago, right after the Fukushima Daiichi nuclear disaster, XKCD released an awesome [Radiation Dose Chart](http://xkcd.com/radiation/). Also around that time portable geiger counters became much easier to get a hold of and fairly cheap. Me being a data nerd; I researched a variety of devices and decided to get a CQ GMC-320. Its an awesome portable geiger counter that has a USB data-port on it that allows you to collect the CPM via a USB serial port. The following Cacti template and Python script is how I generated the following Graph.

![gmc-320](/assets/GMC-320Re.png)

Template: [cacti\_graph\_template\_gmc-320\_-\_cpm\_values.zip](/assets/cacti_graph_template_gmc-320_-_cpm_values.zip)

## Additional GMC-320 Links

- [Amazon Link](https://www.amazon.com/GQ-GMC-320-Plus-Radiation-Detector-equipment/dp/B00I8GQ1EC)
- [User Guide](http://www.gqelectronicsllc.com/gmc320userguide.pdf)
- [Communication Details](http://www.gqelectronicsllc.com/download/GQ-RFC1201.txt)

**This Cacti template will import/update the following items:**

### CDEF

- Trend
- Geiger - sV \[ALL\]
- Geiger - sV \[AVG\]
- Geiger - mR \[ALL\]
- Geiger - mR \[AVG\]

### GPRINT Preset

- Normal
- Exact Numbers
- MicroSievert
- Interger
- MilliRoentgen

### Data Input Method

- GQ - GMC-320Re 3.22

### Data Template

- GQ - CPM

### Graph Template

- GMC-320 - CPM values
