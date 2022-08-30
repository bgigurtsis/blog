---
layout:     post
title:      Security analysis homelab
date:       2022-08-25 23:40:00
summary:    Overview of my security analysis homelab (Splunk/pfSense)
categories: security
comments: true
---
This post will be a very broad overview of my security analysis homelab, including lessons learned and general setup. Please note it is a work in progress and I will update it with more detail as time allows.

![](https://www.bgigurtsis.com/pictures/posts/homelab/NetworkMap.png)

For this setup, I used VMware Workstation as my hypervisor. I chose this over VirtualBox mostly due to my familiarity with VMware Workstation.

### Virtual machine networking

Probably the most important part of the networking setup, and when virtual machine networking 'clicked' for me, was connecting my pfSense VM to my other VM's by adding in their VMnets/virtual switches (`VMnet2/3/4/5/6`):

![](https://www.bgigurtsis.com/pictures/posts/homelab/PfAdapters.jpg)

I also learned about what exactly a [SPAN port](https://www.garlandtechnology.com/tap-vs-span) is, and how I could use that to mirror all the traffic from my victim/vulnerable network into pfSense (`VMnet 3`) and my SecurityOnion instance (`VMnet 5`). The logs from my Windows 2019 server (AD/DC server) were sent to Splunk via a universal forwarder.

### Virtual machine configs

##### pfSense

Below is a screenshot showing my pfSense setup. This includes the IP address mapping to their respective network adapters (`em1/2/3/4/5`). You can further see exactly how it all maps out by seeing where each network adapter (`em1/2/etc...`) lies in the network map at the top of this post (or [click here!](https://www.bgigurtsis.com/pictures/posts/homelab/netpf.png) for easy viewing):

![](https://www.bgigurtsis.com/pictures/posts/homelab/pfsensesetup.PNG)

##### Windows Server 2019 (Active Directory / Domain Controller)

Below shows the very basic setup of my victim network, specifically my AD/DC and one of the Windows 10 machines that I have joined to my domain `billy.local`:

![](https://www.bgigurtsis.com/pictures/posts/homelab/asd.JPG)

##### Splunk

Splunk is the SIEM where I do most of my analysis. Right now it's only set up to receive logs from the universal forwarder I set up on my Windows Server. Below is a basic search, showing all events from the event viewer that indicate a failed login attempt:

![](https://www.bgigurtsis.com/pictures/posts/homelab/splunk.PNG)

Here shows what types of events are being captured from my Windows Server:

![](https://www.bgigurtsis.com/pictures/posts/homelab/wineventlog.PNG)

And finally, the index that I setup to capture the events themselves highlighted at the bottom:

![](https://www.bgigurtsis.com/pictures/posts/homelab/indexes.PNG)  

That's all I have for now! This post is a WIP, and I will be updating it with more detail as time allows.
