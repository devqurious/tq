---
title: "Part 25: Configuring a VIP (DNAT)"
date: 2020-12-28T12:10:51+05:30
thumb_image: "/posts/attachments/dnat.png"
omit_header_text: true
draft: false
tags: ["homecloud", "computers"]
categories: ["HomeCloud"]
---

We [now have three K3s masters](/posts/pi/24_cluster-1/) up an running on three different IPs and but trying to access the apps using any of these IP's defeats the purpose of HA - if that node goes down, so does the IP, and it's all downhill from there. So we need a load balancer, which will have it's own "Virtual" IP, or DNAT IP. 

Time to now change the network architecture again. This is how it now looks like. 

![](/posts/attachments/home_cloud_network.png)

The three nodes in the cluster (newton, coppernicus, and galileo) have now been placed in a separate network called DMZ. This is 172.16.17.x network. 

The wireless devices (clients that will use the application) have been "bridged" using the wifi router onto the 172.16.16.x network. We now have two networks and are ready for the third and final piece of magic. 

Setting a DNAT rule that will translate HTTP/DNS traffic coming from the clients TO the destination gateway IP (https://172.16.16.16)...that single destination will be translated to three destinations in a round robin manner. 

![](/posts/attachments/NAT_rule.png)

(In the picture above, the translated DNAT called Scientists is a list of the IP addresses of...well the scientists). Note the load balancing method as round robin - this is fine for our cluster as all the nodes will sync their state using the embedded etcd cluster we setup in the last post. 

Now we have single IP endpoint for all our nodes and we can use this to access all the apps we will deploy on the cluster. BUT, BUT, BUT. How to test this, as we don't yet have any apps deployed?

Well, luckily for us there is the default Traefik web application listening on all the nodes on the cluster. Let's hit that from one of the clients. 

```
curl -k -vvvv https://192.16.16.16 
```

If all goes well, you should see a 404 error come back. Your client traffic is now being routed to the nodes in the cluster node. Do it a few more times, and watch the round robin going.

{{< youtube  rJsdNwb32Vk >}}

Seeing is believeing. And believing is understanding!

PS: The single point of failure is now the firewall itself. The last bit we need to do is configure the firewalls themselves to be in HA, which will be the topic of a future post.