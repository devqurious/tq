---
title: "Part 17: Traefik - Dashboard (3)"
date: 2020-12-20T12:10:51+05:30
thumb_image: "/posts/attachments/traefik-3.png"
omit_header_text: true
draft: false
tags: ["homecloud", "computers"]
categories: ["HomeCloud"]
---

The power of Traefik is that it can handle multiple web applications. Let's go back to part 9, and reapply the manifest files. 

The manifest yml files we will be using are [here](https://github.com/devqurious/homecloud/tree/main/yml/hello-world)

```
sudo kubectl apply -f mysite-nginx.yml
sudo kubectl create configmap mysite-html --from-file index.html
```
Once the pods are created, see the Traefik dashboard.

![traefik-ingress](/images/pi/traefik-3.png)

Now there are TWO frontend rules and both of them have their own backend pods. The pods are represented as servers, because they are - each of them is serving a different web application!

Pihole at [http://newton/pihole/admin](http://newton/pihole/admin)

and 

Hello at [http://newton/hello-world](http://newton/hello-world)

Now imagine the possibilities - you can deploy as many web apps as you want and traefik can handle the web traffic for all of them.

Neat-o!