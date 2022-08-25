---
layout:     post
title:      Security analysis homelab
date:       2022-08-24 23:40:00
summary:    Setting up my first security analysis homelab (Splunk/pfSense)
categories: security
comments: true
---

This post will be a very broad overview of my security analysis homelab, including lessons learned and general setup.

![](https://www.bgigurtsis.com/pictures/posts/homelab/NetworkMap.png)

For this setup I used VMWare Workstation as my hypervisor. I chose this over VirtualBox mostly due to my familiarity with VMWare Workstation.

### Virtual machine networking


Probably the most important part of the networking setup, and when virtual machine networking 'clicked' for me, was connecting my pfSense VM to my other VM's by adding in their VMnets/virtual switches (VMnet2/3/4/5/6):

![](https://www.bgigurtsis.com/pictures/posts/homelab/PfAdapters.jpg)

I also learned about what exactly a [SPAN port](https://www.garlandtechnology.com/tap-vs-span) is, and how I could use that to mirror all the traffic from my victim/vulnerable network into pfSense (VMnet 3) and my SecurityOnion instance (VMnet 5). The logs from my Windows 2019 server (AD/DC server) were sent to Splunk via a universal forwarder.
