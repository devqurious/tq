---
title: "Part 26: Orchestrations"
date: 2020-12-29T12:10:51+05:30
thumb_image: "/images/pi/iamk3s.png"
omit_header_text: true
draft: false
tags: ["homecloud", "computers"]
categories: ["HomeCloud"]
---

Now that we have re-created our cluster with a 3-node embedded etcd K3s cluster, let's now [re-deploy the hello world application](/posts/pi/9_pihome_hello_world/) and see how it behaves. 

Last time we deployed it on a single node, and so that's where the pod was spun up. But now that we have three nodes, K3s can decide any free node to spin up the pod on. Then, if that node goes down for some reason, K3 will detect that event and then spin up the app again on one of the remaining two healthy nodes. In a bit, your application is back up without any manual intervention.

{{< youtube K0aP1dDJ0QM>}}

Another neat thing K3s can do is spin up as many pods as you want it to - all you have to do is set the replica count in the deployment yml file.

```
spec:
  replicas: 2
```

And you could do this based on load too. 

This is why K3s is called a container orchestrator platform.