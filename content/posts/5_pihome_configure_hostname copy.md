---
title: "Part 5: Configure the hostname"
date: 2020-12-08T08:10:51+05:30
thumb_image: "/posts/attachments/DNS.png"
omit_header_text: true
draft: false
tags: ["homecloud", "computers"]
categories: ["HomeCloud"]
---

We homo sapiens are good with names. Let's give a hostname to our master node. In the network below we have a firewall that can act as a DNS server too, so let's add a DNS host entry to the router. 

![](/posts/attachments/dns_entry.png)

Now anything that's directly connected to the POE switch will see that hostname. It may take a few minutes to take effect. 

![](/posts/attachments/DNS.png)

But what about the devices on the wireless network? Update the devices to use the firewall as the DNS server. And all will be well. Make sure you can now ping and ssh to the master using its hostname. 

```
ping newton
ssh ubuntu@newton
```

I chose the hostname as newton. After all, he really was the master of the 17th century was he not?