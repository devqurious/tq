---
title: "Part 27: A cluster of Piholes"
date: 2020-12-30T12:10:51+05:30
thumb_image: "/images/pi/pi_cluster.jpg"
omit_header_text: true
draft: false
tags: ["homecloud", "computers"]
categories: ["HomeCloud"]
---

Back in [part 14 we installed Pihole on a single node](/posts/pi/14_install-pihole/), now it's time to install it on the 3-node cluster. It's almost the same but with a few caveats. 

### Caveat 1: The DNS Issue

Once Helm installs the Pihole containers, the main Pihole container will be stuck in ImageBackOff stage. This is almost always due to [this](https://github.com/MoJo2600/pihole-kubernetes/issues/88#issuecomment-742276533) issue. Your DNS will stop working on the node where Pihole is being installed and it all goes south from there. To fix it, temporarily, disable systemd-resolved and configure DNS to point to 8.8.8.8 in the /etc/resolve.conf file. 

```
sudo service systemd-resolved stop
sudo vim /etc/resolv.conf
nameserver 8.8.8.8
```

Then re-deploy.

```
sudo kubectl rollout restart deployment pihole -n pihole
```

### Caveat 2: Which node?

Now we have three nodes, where do you make the above changes? Well you can run `-o wide` option to see where Pihole is being installed and fix that node, or just apply the fix on all the nodes. 

### Caveat 3: Which ingress?

The ingress IP is automatically updated when nodes go up and down. If the ingress address is 172.16.17.253 and that node happens to go down, the ingress is automatically updated by K3s to the IP address of one of the other nodes. Which is very cool.

So the admin interface is always available. 

```
ubuntu@newton:~/homecloud/yml/pihole$ sudo kubectl get ing -A
NAMESPACE   NAME                   CLASS    HOSTS                 ADDRESS         PORTS   AGE
default     mysite-nginx-ingress   <none>   *                     172.16.17.254   80      37h
pihole      pihole--ingress        <none>   *                     172.16.17.254   80      10h
```

If .254 went down...then after a while...

```
ubuntu@newton:~/homecloud/yml/pihole$ sudo kubectl get ing -A
NAMESPACE   NAME                   CLASS    HOSTS                 ADDRESS         PORTS   AGE
default     mysite-nginx-ingress   <none>   *                     172.16.17.253   80      37h
pihole      pihole--ingress        <none>   *                     172.16.17.253   80      10h
```

### Last thoughts

Is this Pihole now highly available? The Pihole admin interface is highly available as we can see above, but what about DNS service itself. Let's check.

```
dig @172.16.16.16 google.com

; <<>> DiG 9.10.6 <<>> @172.16.16.16 google.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 29069
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;google.com.			IN	A

;; ANSWER SECTION:
google.com.		8	IN	A	216.58.197.46

;; Query time: 41 msec
;; SERVER: 172.16.16.16#53(172.16.16.16)
;; WHEN: Sat Jan 09 12:06:22 IST 2021
;; MSG SIZE  rcvd: 55
```

With the @, you can force the DNS server that will be used to resolve the name. In this case it is the IP address of the load balancer that has been configured to round robin the request to one of the three nodes in the cluster. And even if you bring one node down, the DNS request should still succeed. 

PS: A handy command to understand DNS is:

```
dig +trace google.com
```

Run this command from one of the clients on the network, and from one of the nodes in the cluster and see how trace changes.