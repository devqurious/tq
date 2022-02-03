---
title: "Part 16: Traefik - The Dashboard (2)"
date: 2020-12-19T12:10:51+05:30
thumb_image: "/posts/attachments/traefik-post.png"
omit_header_text: true
draft: false
tags: ["homecloud", "computers"]
categories: ["HomeCloud"]
---

List all the ingress by running the following command 

```
sudo kubectl get ing -A -o wide
```

The use the `sudo kubectl delete ing <name> -n <namespace>` command to delete the ingresses one by one. Watch the traefik get cleared off, and soon, all you have is...nothing!

![traefik-empty](/images/pi/traefik-empty.png)

Now try to access [http://newton/pihole/admin](http://newton/pihole/admin) as before. You get a 404 error. This is traefik telling you that it could not find the page you are looking for. 

Let's fix that again. Let's go back and re-create the ingress route using [this](https://github.com/devqurious/homecloud/blob/main/yml/pihole/pihole-web.yml) manifest. 

```
ubuntu@newton:~/homecloud/yml/pihole$ sudo kubectl apply -f pihole-web.yml 
ingress.networking.k8s.io/pihole--ingress created
```

![pihole-traefik](/images/pi/pihole-traefik.png)

The providers tab displays the frontend rules that have been configured in Traefik, and which backends pods that handle the requests. 

![pihole-traefik](/images/pi/traefik-dashboard.png)

The health dashboard also the dynamic nature of the traefik proxy. How many requests it received. How many it could handle successfully (HTTP Status 200) and how many requests it could not (HTTP Status 404, and others)...

It also displays how quickly it was able to handle those request - an important metric that will indicate how happy, or unhappy your customers are. 

Quite magical!
