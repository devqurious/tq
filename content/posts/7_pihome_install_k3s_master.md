---
title: "Part 7: Install K3s on master node"
date: 2020-12-10T08:10:51+05:30
thumb_image: "/images/pi/K3s.png"
omit_header_text: true
draft: false
tags: ["homecloud", "computers"]
categories: ["HomeCloud"]
---

Finally, we're ready to install K3s on the master node. First step is to update a /boot/firmware/cmdline.txt (*not* /boot/cmdline.txt) as some sites mention. Add the first two parameters (cgroup_memory and cgroup_enable) as shown below.

```
cgroup_memory=1 cgroup_enable=memory net.ifnames=0 dwc_otg.lpm_enable=0 console=serial0,115200 console=tty1 root=LABEL=writable rootfstype=ext4 elevator=deadline rootwait fixrtc
```

Reboot, and then run the following command `cat /proc/cmdline` to verify that the updated parameters are in effect.

```
ubuntu@ubuntu:~$ cat /proc/cmdline 
 coherent_pool=1M 8250.nr_uarts=1 snd_bcm2835.enable_compat_alsa=0 snd_bcm2835.enable_hdmi=1 snd_bcm2835.enable_headphones=1 bcm2708_fb.fbwidth=0 bcm2708_fb.fbheight=0 bcm2708_fb.fbswap=1 smsc95xx.macaddr=DC:A6:32:D4:4D:B2 vc_mem.mem_base=0x3ec00000 vc_mem.mem_size=0x40000000  cgroup_memory=1 cgroup_enable=memory net.ifnames=0 dwc_otg.lpm_enable=0 console=ttyS0,115200 console=tty1 root=LABEL=writable rootfstype=ext4 elevator=deadline rootwait fixrtc quiet splash
```

Now install K3s. 

```
curl -sfL https://get.k3s.io | sh -
```

Amazingly, that's all there is to it. The above command will download the stable version of K3s and then run a script to install it on the pi. Once it is done, verify:

```
ubuntu@ubuntu:~$ sudo kubectl get nodes
NAME     STATUS   ROLES    AGE    VERSION
ubuntu   Ready    master   133m   v1.19.4+k3s1
```

Running pods. 

```
ubuntu@newton:~/homecloud/yml$ sudo kubectl get pods -o wide --all-namespaces
NAMESPACE     NAME                                     READY   STATUS      RESTARTS   AGE     IP          NODE     NOMINATED NODE   READINESS GATES
kube-system   metrics-server-7b4f8b595-phxs8           1/1     Running     0          2m42s   10.42.0.2   newton   <none>           <none>
kube-system   local-path-provisioner-7ff9579c6-sv7rh   1/1     Running     0          2m42s   10.42.0.4   newton   <none>           <none>
kube-system   coredns-66c464876b-2wnnl                 1/1     Running     0          2m42s   10.42.0.5   newton   <none>           <none>
kube-system   helm-install-traefik-mpfjq               0/1     Completed   0          2m42s   10.42.0.3   newton   <none>           <none>
kube-system   svclb-traefik-jh45j                      2/2     Running     0          97s     10.42.0.6   newton   <none>           <none>
kube-system   traefik-5dd496474-hxczn                  1/1     Running     0          97s     10.42.0.7   newton   <none>           <none>
```

Running services.

```
ubuntu@newton:~/homecloud/yml$ sudo kubectl get services -o wide --all-namespaces
NAMESPACE     NAME                 TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)                      AGE     SELECTOR
default       kubernetes           ClusterIP      10.43.0.1      <none>         443/TCP                      3m30s   <none>
kube-system   kube-dns             ClusterIP      10.43.0.10     <none>         53/UDP,53/TCP,9153/TCP       3m28s   k8s-app=kube-dns
kube-system   metrics-server       ClusterIP      10.43.19.110   <none>         443/TCP                      3m28s   k8s-app=metrics-server
kube-system   traefik-prometheus   ClusterIP      10.43.75.178   <none>         9100/TCP                     2m10s   app=traefik,release=traefik
kube-system   traefik              LoadBalancer   10.43.8.254    172.16.16.31   80:31829/TCP,443:32207/TCP   2m10s   app=traefik,release=traefik
```

Running ingresses. 

```
ubuntu@newton:~/homecloud/yml$ sudo kubectl get ing -o wide --all-namespaces
Warning: extensions/v1beta1 Ingress is deprecated in v1.14+, unavailable in v1.22+; use networking.k8s.io/v1 Ingress
No resources found
```

That's a lot that got un-packed. 

To uninstall:

```
sudo /usr/local/bin/k3s-uninstall.sh
```
