---
title: "Part 32: Exposing a service using host ports"
date: 2021-01-04T12:10:51+05:30
thumb_image: "/posts/attachments/hostport.jpg"
omit_header_text: true
draft: false
tags: ["homecloud", "computers"]
categories: ["HomeCloud"]
---

One way of using services that reside in a cluster is by [exposing them on a port on the host](https://rancher.com/docs/rancher/v2.x/en/v1.6-migration/expose-services/). 

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hc-rsyncer-app
  labels:
    name: hc-rsyncer-app
spec:
  replicas: 1
  selector:
    matchLabels:
      name: hc-rsyncer-app
  template:
    metadata:
      labels:
        name: hc-rsyncer-app
    spec:
      containers:
        - name: hc-rsyncer-app
          image: drathaqm/home_sync
          imagePullPolicy: Always
          ports:
            - containerPort: 22
              hostPort: 9890
              name: 80tcp9890
              protocol: TCP
```

Notice the *ports* portion of the deployment manifest now. Traffic on port 9890 on the host will be forwarded to port 22 running inside our container. But which host? Aha - we can only know that once we have deployed our cluster. So deploy it using `apply` command, then check where the container is deployed. 

```
sudo kubectl get pods -o wide
NAME                             READY   STATUS    RESTARTS   AGE   IP            NODE         NOMINATED NODE   READINESS GATES
hc-rsyncer-app-b5ccd8c8d-n8xkm   1/1     Running   0          10h   10.42.2.125   copernicus   <none>           <none>
mysite-nginx-5559ffd776-nlg69    1/1     Running   3          13d   10.42.1.144   galileo      <none>           <none>
```

Now you know that the host on which port forwarding of 9890 to 22 is on....copernicus! So to access this service, you have to use the IP address of the copernicus node in the cluster. You can verify it easily:

On the mac (172.16.17.253 is the IP address of copernicus node):

```
nc 172.16.17.253 9890
SSH-2.0-OpenSSH_8.2p1 Ubuntu-4ubuntu0.1
```

But here lies one of [the biggest disadvantages](https://rancher.com/docs/rancher/v2.x/en/v1.6-migration/expose-services/#hostport-cons) of this approach - what happens when copernicus goes down? 

Clients have been using it's IP address to connect to the service, and boom, now suddenly they have lost that service. Also, when the nodes reboot, it's possible that K3s might launch the service on another node (newton, say) and now all the clients will need to be re-configured to use a new IP address. 

This configuration also does not sit well with our load balancer. It's only every third request that will work! After all, that's exactly what round robin is all about. 

```
devendrarath@mini2020 qblog % nc -vz 172.16.16.16 30036
Connection to 172.16.16.16 port 30036 [tcp/*] succeeded!
devendrarath@mini2020 qblog % nc -vz 172.16.16.16 30036
nc: connectx to 172.16.16.16 port 30036 (tcp) failed: Connection refused
devendrarath@mini2020 qblog % nc -vz 172.16.16.16 30036
nc: connectx to 172.16.16.16 port 30036 (tcp) failed: Connection refused
devendrarath@mini2020 qblog % nc -vz 172.16.16.16 30036
Connection to 172.16.16.16 port 30036 [tcp/*] succeeded!
devendrarath@mini2020 qblog % nc -vz 172.16.16.16 30036
nc: connectx to 172.16.16.16 port 30036 (tcp) failed: Connection refused
devendrarath@mini2020 qblog % nc -vz 172.16.16.16 30036
nc: connectx to 172.16.16.16 port 30036 (tcp) failed: Connection refused
```

Painful, a bit. (*It builds character, you say?*) 