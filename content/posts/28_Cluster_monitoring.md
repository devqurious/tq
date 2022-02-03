---
title: "Part 28: Monitoring the cluster"
date: 2020-12-31T12:10:51+05:30
thumb_image: "/posts/attachments/grafana.png"
omit_header_text: true
draft: false
tags: ["homecloud", "computers"]
categories: ["HomeCloud"]
---

Jeff Geerling [has a great post](https://www.jeffgeerling.com/blog/2020/raspberry-pi-cluster-episode-4-minecraft-pi-hole-grafana-and-more) on how to monitor your K3s nodes and it works beautifully. Of coure, he uses the work of [this repo](https://github.com/carlosedp/cluster-monitoring) and thus mankind plods on. 

The main thing to note is the IP of the master node. In our case, it is the VIP (or IP address of the load balancer).

```
k3s: {
    enabled: true,
    master_ip: ['172.16.16.16'],
  },

  // Domain suffix for the ingresses
  suffixDomain: '172.16.16.16.nip.io',
```

Now, watch the grafana in all it's glory. The default username/password is admin/admin.

![](/posts/attachments/grafana.png)