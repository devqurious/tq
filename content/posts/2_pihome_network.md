---
title: "Part 2: Lay of the land"
date: 2020-12-05T08:10:51+05:30
thumb_image: "/posts/attachments/HomeCloud.png"
omit_header_text: true
draft: false
tags: ["homecloud"]
categories: ["HomeCloud"]
---

![](/posts/attachments/HomeCloud.png)

It's a good idea to visualize where the cloud will sit among other things. We will start with one RPi box as the master. It will be controlled by a Macbook Pro - yes, there are no absolute masters in life. :)

The firewall here is pretty central to the network. It's where we configure DHCP, DNS and a whole lot more. Without it, setting up this network would be downright painful.

This is the time to also note down the IP addresses of your devices. If you have multiple wired and wireless neteorks make sure you can ping across them. 

Onward!

NOTE: As things progressed, it [became evident](/posts/pi/18_pihole-parental-control-1) that the better place to place the Pi was in the wifi network itself, for better monitoring and device discovery. So the network was modified to the following:

![](/posts/attachments/HomeCloud-v2.png)
