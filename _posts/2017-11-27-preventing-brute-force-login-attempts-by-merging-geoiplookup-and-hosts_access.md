---
title: 'Preventing brute force login attempts by merging geoiplookup and hosts_access'
date: '2017-11-27T12:55:03+00:00'
author: deaves
layout: post
permalink: /2017/11/preventing-brute-force-login-attempts-by-merging-geoiplookup-and-hosts_access/
categories:
    - Ansible
    - Linux
    - 'Linux Admin'
    - 'Linux Scripts'
    - 'Linux Security'
    - Networking
tags:
    - aclexec
    - ansible
    - geoip
    - geoiplookup
    - hosts.allow
    - 'linux acl'
    - 'tcp wrappers'
---

To me networking and UNIX run hand-in-hand; as I have been getting into [Ansible](https://www.ansible.com/) to do network automation I am also using it to automate and enforce consistency between all of my other hosts. The following is a simple playbook to configure Debian based hosts to perform a geoiplookup against all incoming hosts to prevent brute force login attempts using host\_access. I have to admit the original script and idea are not mine; they originated from the following blog post: [Limit your SSH logins using GeoIP](https://www.axllent.org/docs/view/ssh-geoip). As more and more systems are migrated into cloud environments automating and enforcing security controls like this one are of critical importance.

## geowrapper.yaml: a /usr/local/bin/ansible-playbook script, ASCII text executable

```yaml
#!/usr/local/bin/ansible-playbook
## Configure Debian OS family to geoiplookup against all incoming hosts.
## 2017 (v.01) - Playbook from www.davideaves.com
---
- name: GEO Wrapper
  hosts: all
  become: yes
  gather_facts: yes
  tags: host_access

  vars:
    geocountries: "US"
    geofilter: "/opt/geowrapper.sh"

  tasks:
  - name: "Fail if OS family not Debian"
    fail:
      msg: "Distribution not supported"
    when: ansible_os_family != "Debian"

  - name: "Fetch geoip packages"
    apt:
      name:
        - geoip-bin
        - geoip-database
      state: latest
      update_cache: yes
    register: geoip
    when: ansible_os_family == "Debian"

  - name: "Wrapper script {{ geofilter }}"
    copy:
      content: |
        #!/bin/bash
        # Ansible Managed: GeoIP aclexec script for Linux TCP wrappers.
        ## Source: http://www.axllent.org/docs/view/ssh-geoip

        # UPPERCASE space-separated country codes to ACCEPT
        ALLOW_COUNTRIES="{{ geocountries }}"

        if [ $# -ne 1 ]; then
          echo "Usage:  `basename $0` ip" 1>&2
          exit 0 # return true in case of config issue
        fi

        COUNTRY=`/usr/bin/geoiplookup $1 | awk -F ": " '{ print $2 }' | awk -F "," '{ print $1 }' | head -n 1`

        [[ $COUNTRY = "IP Address not found" || $ALLOW_COUNTRIES =~ $COUNTRY ]] && RESPONSE="ALLOW" || RESPONSE="DENY"

        if [ $RESPONSE = "ALLOW" ]
        then
          exit 0
        else
          logger "$RESPONSE connection from $1 ($COUNTRY)"
          exit 1
        fi
      dest: "{{ geofilter }}"
      mode: "0755"
      owner: root
      group: root
    ignore_errors: yes
    register: geowrapper
    when: geoip|success

  - name: "Mappings in /etc/hosts.allow"
    blockinfile:
      path: /etc/hosts.allow
      state: present
      content: |
        ALL: 10.0.0.0/8
        ALL: 172.16.0.0/12
        ALL: 192.168.0.0/16
        ALL: ALL: aclexec {{ geofilter }} %a
    when: geowrapper|success

  - name: "Mappings in /etc/hosts.deny"
    blockinfile:
      path: /etc/hosts.deny
      state: present
      content: "ALL: ALL"
    when: geowrapper|success
```
