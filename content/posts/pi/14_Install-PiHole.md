---
title: "Part 14: Installing Pihole"
date: 2020-12-17T12:10:51+05:30
thumb_image: "/images/pi/pihole-logo.png"
omit_header_text: true
draft: false
tags: ["homecloud", "computers"]
categories: ["HomeCloud"]
---

Let's use Helm now to install something a bit more complex - the [pihole DNS server](https://pi-hole.net). This provides ad-blocking. It's pretty neat. 

As usual.

```
helm repo add mojo2600 https://mojo2600.github.io/pihole-kubernetes/
helm repo update
```

We will install pihole in it's own namespace

```
sudo kubectl create namespace pihole
```

When installing an app via Helm, it is common to provide that app some defaults. This is passed in via a [defaults file](https://artifacthub.io/packages/helm/mojo2600/pihole). For pihole, [this](https://github.com/devqurious/homecloud/blob/main/yml/pihole/ph_values.yml) is what the defaults file contains.

```
---
persistentVolumeClaim:
  enabled: true
ingress:
  enabled: true
serviceWeb:
  type: ClusterIP
serviceDns:
  loadBalancerIP: '172.16.16.31'
  annotations:
    metallb.universe.tf/allow-shared-ip: pihole-svc
  type: LoadBalancer
resources:
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi
# If using in the real world, set up admin.existingSecret instead.
adminPassword: admin
```

So create a file called pihole_values.yml and pass it via the --values parameter. Now, you can install the app like so:

```
ubuntu@newton:~/homecloud/yml/pihole$ helm install --version '1.8.22' --namespace pihole --values ph_values.yml pihole mojo2600/pihole
NAME: pihole
LAST DEPLOYED: Fri Jan  8 11:37:29 2021
NAMESPACE: pihole
STATUS: deployed
REVISION: 1

```

This installs the pihole application itself, and three services as shown below. 

```
NAMESPACE     NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)                      AGE
default       kubernetes             ClusterIP      10.43.0.1       <none>         443/TCP                      37h
kube-system   kube-dns               ClusterIP      10.43.0.10      <none>         53/UDP,53/TCP,9153/TCP       37h
kube-system   metrics-server         ClusterIP      10.43.19.110    <none>         443/TCP                      37h
kube-system   traefik-prometheus     ClusterIP      10.43.75.178    <none>         9100/TCP                     37h
kube-system   traefik                LoadBalancer   10.43.8.254     172.16.16.31   80:31829/TCP,443:32207/TCP   37h
default       pihole-web             ClusterIP      10.43.106.80    <none>         80/TCP,443/TCP               9h
default       pihole-dns-tcp         LoadBalancer   10.43.184.128   172.16.16.31   53:30466/TCP                 9h
default       pihole-dns-udp         LoadBalancer   10.43.254.147   172.16.16.31   53:31929/UDP,67:30512/UDP    9h
default       mysite-nginx-service   ClusterIP      10.43.33.101    <none>         80/TCP                       9h
```

If you now try to `curl http://newton` from outside the cluster (say, from your mac) you will get a 404 error. Because the default ingress controller treafik does not know where to forward the request to. 

Now the final bit is to create an ingress path to the service so that external clients can access the web service (needed for administration). For that we need [a second yaml](https://github.com/devqurious/homecloud/blob/main/yml/pihole/ph_web.yml) file.

```
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: pihole--ingress
  annotations:
    kubernetes.io/ingress.class: "traefik"
    traefik.frontend.rule.type: PathPrefixStrip
spec:
  rules:
  - http:
      paths:
      - path: /pihole
        pathType: Prefix
        backend:
          service:
            name: pihole-web
            port:
              number: 80
```

Note the kind - Ingress. Also note that we're exposing the service on the /pihole path. Finally, note the service name as pihole-web, which should exactly match the name of the service that was create in the previous step. Now apply this manifest.

```
sudo kubectl apply -f pihole-web.yml -n pihole
```

Now you should be able to open the wonderful web UI of pihole by navigating to [http://newton/pihole/admin/](http://newton/pihole/admin/). 

![](/images/pi/pi_home.png)

### The w(hole) picture

Now a lot of resources were installed in this post. It helps to have a mental model of all the various moving parts of the pihole deployment.

![](/images/pi/pihole_overview.jpg)

```
sudo kubectl get ing -A
```

At the entrance of the cluster is the Traefik reverse proxy. Any client that wants to get into the cluster basically connects to this reverse proxy, and it is this reverse proxy that then connects back to the actual application - in this case, the pihole app. 

How does Traefik know where to route? In the spec rules, and in this case, it uses the path prefix (/pihole) to forward the request to the pihole service. 

```
sudo kubectl get svc - A
```

Why service? Well, there could be multiple pihole pods, and it would not who to forward the request to. Instead, it knows there is one service pod that is responsible for all requests to /pihole. This service, then, is the one that receives the requests. But how does it know which pihole pod to send the request to? 

```
sudo kubectl get pod -A
```

The service uses the app selector, which is part of it's configuration, to find out the pihole pod it needs to send the request to. If there are mutliple pods, it will distribute the requests evenly across them. Finally the pihole web application receives the request when it reaches the pihole pod. And that's how an admin see the pihole web administration page.

### Upgrading Pihole

New bad domains pop up all the time, and luckily PiHole [updates it's list of bad domains automatically]
9https://discourse.pi-hole.net/t/does-pi-hole-keep-the-lists-updated/13224), once a week (I think). You can see the last updated timestamp on the Dashboards page by hovering on the "Blocked Domains" panel. You can update this manually too, by clicking on "Tools | Update Gravity."

From time to time, you might of course want to update Pihole app itself. To do so, first check the version you have:

```
sudo helm -n pihole list
NAME  	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART        	APP VERSION
pihole	pihole   	1       	2022-01-26 12:40:33.399618712 +0000 UTC	deployed	pihole-1.8.22	5.2    
```

Now, run the upgrade.To find out the latest version go to the [homepage](https://artifacthub.io/packages/helm/mojo2600/pihole) of the repo. Currently, the latest version is 2.5.4.

```
helm upgrade --version 2.5.4 pihole mojo2600/pihole -n pihole
NAME  	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART       	APP VERSION
pihole	pihole   	2       	2022-01-29 05:32:52.867983149 +0000 UTC	deployed	pihole-2.5.4	2021.10.1  
```

And voila, just like that, you're on the latest version.

Test:

![Example image](https://embed.creately.com/w8MaxnnCPjZ?type=svg)

### The DNS issue

Check if pihole is actually running or not by running the following command:

```
sudo kubectl get pods -A
pihole        pihole-7f47c6fc8b-lh6gf                           0/1     Pending            0          11m
```

If the status is Pending, then it's a problem.

Pihole installs it's own DNS service which may conflict with the DNS settings in the base OS - Ubuntu 20.04 in this case. Try setting it to 127.0.0.1 if you're facing problems getting pihole to run. Setting DNS in Ubuntu is convoluted, and there are a 101 different mechanisms noted in Stack Overflow. Try the following:

```
sudo service systemd-resolved stop
sudo vim /etc/systemd/resolved.conf

[Resolve]
DNS=127.0.0.1
#FallbackDNS=
#Domains=
#LLMNR=no
#MulticastDNS=no
#DNSSEC=no
#DNSOverTLS=no
#Cache=no-negative
#DNSStubListener=yes
#ReadEtcHosts=yes

sudo service systemd-resolved start
systemd-resolve --status
```

Alternately, you can also try setting the DNS server at the interface level using netplan.

```
sudo vim /etc/netplan/01-netcfg.yaml 

network:
    ethernets:
        eth0:
            dhcp4: false
            addresses: [172.16.16.31/24]
            gateway4: 172.16.16.16
            nameservers:
                    addresses:  [127.0.0.1]
            optional: true
    version: 2

sudo netplan apply
```

PS: You may also need to just temporarily disable systemd-resolvd and manually set the DNS nameserver to 8.8.8.8 in /etc/hosts file for the name resolution to work while Pihole is getting installed. After that, you can restart the systemd-resolvd service,  to go back to original settings. 

```
sudo service systemd-resolved stop
sudo vim /etc/resolve.conf
nameserver 8.8.8.8
```

The problem is that on every reboot, you will end up in this chicken and egg case, where, for sometime  you have to allow pihole to use /etc/resolv.conf before it can start using it's own DNS service. This is a TBD problem. 

In all cases, you need to make sure you are able to ping wwww.google.com from the node(s).

## If you are deploying pihole in a cluster...

Remember to do these commands on the node where the main pihole container is being installed... (use the -o wide command to find that out.)

## References:
https://levelup.gitconnected.com/a-guide-to-k3s-ingress-using-traefik-with-nodeport-6eb29add0b4b
https://medium.com/@geraldcroes/kubernetes-traefik-101-when-simplicity-matters-957eeede2cf8
