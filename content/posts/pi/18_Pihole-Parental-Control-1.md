---
title: "Part 18: Pihole Parental Control - 1"
date: 2020-12-21T12:10:51+05:30
thumb_image: "/images/pi/parental-control.png"
omit_header_text: true
draft: false
tags: ["homecloud", "computers"]
categories: ["HomeCloud"]
---

Pihole is a great adblocker. But can it do its bit for parental control?

The problem with the original network design (Part 2, lay of the land) was that all the requests to pihole were coming in from the firewall, so it was not possible to segregate the traffic from the kid's machine vs traffic from the other machines on the network. So a network re-design was needed which brought in the pihole inside the wireless network where all the machines lived. 

![](/images/pi/HomeCloud-v2.png)

Once this is done, you should be able to see the client that you wish to control, in the "Top Clients" dashboard widget. 

![](/images/pi/top-clients.png)

Now group based configuration should work.