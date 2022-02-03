---
title: "Part 8: Kubectl"
date: 2020-12-11T08:10:51+05:30
thumb_image: "/images/pi/controller.jpg"
omit_header_text: true
draft: false
tags: ["homecloud", "computers"]
categories: ["HomeCloud"]
---

We have K3s up and the all powerful *kubectl* command line utility. 

Check that the systemd service started correctly.

```
sudo systemctl status k3s
```

List all the running pods.

```
ubuntu@newton:~$ sudo kubectl get pods --all-namespaces -o wide
NAMESPACE     NAME                                     READY   STATUS              RESTARTS   AGE   IP          NODE     NOMINATED NODE   READINESS GATES
kube-system   helm-install-traefik-pvhrq               0/1     ContainerCreating   0          58s   <none>      newton   <none>           <none>
kube-system   local-path-provisioner-7ff9579c6-cfpv5   1/1     Running             0          58s   10.42.0.5   newton   <none>           <none>
kube-system   metrics-server-7b4f8b595-shh8s           1/1     Running             0          58s   10.42.0.3   newton   <none>           <none>
kube-system   coredns-66c464876b-cn5r4                 1/1     Running             0          58s   10.42.0.2   newton   <none>           <none>
```

There's a lot to unpack there. 

Namespaces. Nodes. And what *are* those IP addresses? 

