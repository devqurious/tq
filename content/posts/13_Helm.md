---
title: "Part 13: Installing Helm"
date: 2020-12-16T16:10:51+05:30
thumb_image: "/images/pi/boat.png"
omit_header_text: true
draft: false
tags: ["homecloud", "computers"]
categories: ["HomeCloud"]
---

Installing apps using the `kubectl apply -f` option quickly becomes tedious. So we have *Helm* package manager for installing apps into the K3s cluster. 

First, we have to [install Helm](https://helm.sh/docs/intro/install/). Choose to install via script.

```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

Helm installs to /usr/local/bin, and you ever need to uninstall it, simply delete the directory.

```
ubuntu@newton:~/homecloud/yml$ ./get_helm.sh 
Downloading https://get.helm.sh/helm-v3.4.2-linux-arm64.tar.gz
Verifying checksum... Done.
Preparing to install helm into /usr/local/bin
[sudo] password for ubuntu: 
helm installed into /usr/local/bin/helm
```

Check the Helm version. We have version 3.4.2. 

```
ubuntu@newton:~/homecloud/yml$ helm version --short
v3.4.2+g23dd3af
```

Add a repo that Helm can use to search and install apps.

```
helm repo add stable https://charts.helm.sh/stable
```

Try to [list all the releases](https://helm.sh/docs/helm/helm_list/) in the system. This should be an empty list as we've not used Helm to install anything. 

```
ubuntu@newton:~/homecloud/yml$ helm ls
Error: Kubernetes cluster unreachable: Get "http://localhost:8080/version?timeout=32s": dial tcp 127.0.0.1:8080: connect: connection refused
```

Try to [install an app](https://helm.sh/docs/intro/quickstart/#initialize-a-helm-chart-repository) 

```
ubuntu@newton:~/homecloud/yml$ helm install stable/mysql --generate-name
WARNING: This chart is deprecated
Error: Kubernetes cluster unreachable: Get "http://localhost:8080/version?timeout=32s": dial tcp 127.0.0.1:8080: connect: connection refused

```

Houston, we have a problem. 

## .kube permissions

It turns out we need to [create a new config file](https://stackoverflow.com/questions/45914420/why-tiller-connect-to-localhost-8080-for-kubernetes-api) in our user's home directory for helm to work correctly, with the right permissions.

```
mkdir ~/.kube
sudo kubectl config view --raw > ~/.kube/config
chmod 400 /home/ubuntu/.kube/config
```

Now confirm the permissions of the .kube dir.

```
drwxrw---- 3 ubuntu ubuntu  4096 Jan  8 16:17 .kube
```

```
ubuntu@newton:~/.kube$ ls -al
total 16
drwxrw---- 3 ubuntu ubuntu 4096 Jan  8 16:17 .
drwxr-xr-x 7 ubuntu ubuntu 4096 Jan  9 03:50 ..
drwxrw---- 4 ubuntu ubuntu 4096 Jan  8 16:17 cache
-rw--w---- 1 ubuntu ubuntu 2957 Jan  8 14:42 config
```

(Ignore the cache dir if its not there - that will get created as you run helm commands which install packages.)

See [this](https://github.com/helm/helm/issues/8776#issuecomment-742607909) for more information. And [this](https://github.com/charmed-kubernetes/bundle/issues/173).

## Getting back

Now, re-run the command.

```
ubuntu@newton:~/homecloud/yml$ helm ls
WARNING: Kubernetes configuration file is group-readable. This is insecure. Location: /home/ubuntu/.kube/config
WARNING: Kubernetes configuration file is world-readable. This is insecure. Location: /home/ubuntu/.kube/config
NAME	NAMESPACE	REVISION	UPDATED	STATUS	CHART	APP VERSION
```

Nothing listed, but hey, Helm is now working. Let's try to install something.

```
helm install stable/mysql --generate-name
```

```
ubuntu@newton:~/homecloud/yml$ helm ls
WARNING: Kubernetes configuration file is group-readable. This is insecure. Location: /home/ubuntu/.kube/config
WARNING: Kubernetes configuration file is world-readable. This is insecure. Location: /home/ubuntu/.kube/config
NAME            	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART      	APP VERSION
mysql-1608355784	default  	1       	2020-12-19 05:29:58.264871295 +0000 UTC	deployed	mysql-1.6.9	5.7.30 
```

Nice - something got installed. 

```
ubuntu@newton:~/homecloud/yml$ helm ls
NAME            	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART      	APP VERSION
mysql-1608355784	default  	1       	2020-12-19 05:29:58.264871295 +0000 UTC	deployed	mysql-1.6.9	5.7.30     
```

All well, now.

But let's get rid of tha that app, since we don't really want it. 

```
ubuntu@newton:~/homecloud/yml$ helm uninstall mysql-1608355784
release "mysql-1608355784" uninstalled
```

```
ubuntu@newton:~/homecloud/yml$ sudo kubectl get pods --all-namespaces
NAMESPACE     NAME                                     READY   STATUS      RESTARTS   AGE
kube-system   metrics-server-7b4f8b595-cdp5k           1/1     Running     1          6d17h
kube-system   local-path-provisioner-7ff9579c6-g7cmx   1/1     Running     1          6d17h
kube-system   svclb-traefik-nsbqr                      2/2     Running     2          6d17h
kube-system   coredns-66c464876b-s4m9f                 1/1     Running     1          6d17h
kube-system   traefik-5dd496474-68m46                  1/1     Running     1          6d17h
```

All well, *now*.

