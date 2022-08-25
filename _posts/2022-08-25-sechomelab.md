---
layout:     post
title:      Security analysis homelab
date:       2022-08-24 23:40:00
summary:    Setting up my first security analysis homelab (Splunk/pfSense)
categories: security
comments: true
---

This post will go into a very broad overview of the homelab I setup in order to learn a multitude of things from security analysis to networking with several virtual machines (VM's).

![](https://www.bgigurtsis.com/pictures/posts/homelab/NetworkMap.png

For this setup I used VMWare Workstation as my hypervisor. I chose this over VirtualBox mostly due to my familiarity with VMWare Workstation.

### Virtual machine networking

Probably the most important part of the networking setup, and when virtual machine networking 'clicked' for me, was connecting my pfSense VM to my other VM's by adding in their VMnets/virtual switches (VMnet2/3/4/5/6):

![](https://www.bgigurtsis.com/pictures/posts/homelab/pfadapters.png
