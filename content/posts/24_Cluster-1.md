---
title: "Part 24: 3-Node cluster"
date: 2020-12-27T12:10:51+05:30
thumb_image: "/posts/attachments/IMG_6442.JPG"
omit_header_text: true
draft: false
tags: ["homecloud", "etcd"]
categories: ["HomeCloud"]
---

Stuff is rolling along just fine, but we have one problem. 

```
ubuntu@ubuntu:~$ sudo kubectl get nodes -A
NAME     STATUS   ROLES    AGE   VERSION
ubuntu   Ready    master   23s   v1.19.5+k3s2
```

We have only one master node, and if that were to go down, our minecraft gamers would be very unhappy. Let's make a [3 node HA embedded cluster](https://rancher.com/docs/k3s/latest/en/installation/ha-embedded/). 

# Prepare the pies

![](/images/pi/IMG_6442.JPG)

Let's restart with 3 pi's this time. [Install the OS](/posts/pi/3_pihome_install_os/) and configure them all with three static IP addresses.Then, [install K3s](/posts/pi/7_pihome_install_k3s_master/) on all the nodes.

# Set hostnames

[Set hostnames](https://linuxize.com/post/how-to-change-hostname-on-ubuntu-18-04/) properly. Easier to remember when you're juggling three servers initially. In my case, without setting the hostnames of the individual pi's I could NOT form the cluster!

Also, without proper hostnames in place, the cluster formation does not take place correctly. So `hostnamectl` away!

# Switch to manual 

Login in to each one of them and stop the K3s service. We will run K3s manually first which makes it easier to debug problems when the cluster is being formed. 

```
sudo systemctl stop k3s.service
```

Then on the first node:

```
sudo k3s server --token foo --cluster-init
```

If you now open another SSH session and run the following command you will see the first node is up and running with an additional role (etcd)

```
ubuntu@ubuntu:~$ sudo kubectl get nodes
NAME     STATUS   ROLES         AGE    VERSION
ubuntu   Ready    etcd,master   115s   v1.19.5+k3s2
```

Now, SSH into the second node and try to join the first node. Remember to give the same token ("foo", in this case). Here 192.168.1.254 is the IP address of the first node which was started with the cluster --init option.

(On node 2)
```
sudo k3s server --server https://192.168.1.254:6443 --token foo
```

Now run the command on either of the two nodes.

```
ubuntu@ubuntu:~$ sudo kubectl get nodes
NAME          STATUS   ROLES         AGE   VERSION
coppernicus   Ready    etcd,master   22m   v1.19.5+k3s2
newton        Ready    etcd,master   24m   v1.19.5+k3s2
```

Two master nodes are up. Repeat for the third node, and soon you will have the entire cluster up and running. Awesome, possum.

```
ubuntu@ubuntu:~$ sudo kubectl get nodes
NAME          STATUS   ROLES         AGE   VERSION
coppernicus   Ready    etcd,master   22m   v1.19.5+k3s2
galileo       Ready    etcd,master   22s   v1.19.5+k3s2
newton        Ready    etcd,master   24m   v1.19.5+k3s2
```

# Switch to automatic

Once the basic cluster is working, it's time to tweak the configuration so that the cluster is formed automatically on reboot. 

On the node that will initiate the cluster...

```
sudo vim /etc/systemd/system/k3s.service
...
ExecStart=/usr/local/bin/k3s \
    server \
    --token foo \
    --cluster-init \
```

```
sudo systemctl daemon-reload
sudo systemctl restart k3s.service
```

On the other two nodes...

```
sudo vim /etc/systemd/system/k3s.service
...
ExecStart=/usr/local/bin/k3s \
    server \
    --server https://192.168.1.254:6443 \
    --token foo \
```

```
sudo systemctl daemon-reload
sudo systemctl restart k3s.service
```

Reboot and watch the cluster form automatically. You're well on your way to HA!


## Troubleshooting

Sometimes it's worth cleaning up completely and trying again. Warning: This will remove all container pods you may have created

```
sudo rm -rf /var/lib/rancher/k3s/
```

After this, try starting k3s manually from the command line and check for errors. If all is well, you should see all the three nodes go into a healthy state. 


Of you can go one level deeper and un-install k3s to start from the beginning. 


```
sudo /usr/local/bin/k3s-uninstall.sh
sudo rm -rf /var/lib/rancher
sudo rm -rf /etc/rancher
```


