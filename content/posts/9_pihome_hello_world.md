---
title: "Part 9: Hello World"
date: 2020-12-12T08:10:51+05:30
thumb_image: "/posts/attachments/hello-world.png"
omit_header_text: true
draft: false
tags: ["homecloud", "computers"]
categories: ["HomeCloud"]
---

It all starts [here](https://en.wikipedia.org/wiki/%22Hello,_World!%22_program). 

Ultimately, all of this is about deploying apps. So let's deploy one, shall we? 

We will deploy the nginx app, a web server that will return a page that has one message on it - Hello, World (surprise, surprise). There are three thing we need:

- The nginx app definition (the app will be pulled from dockerhub)
- A service definition 
- An ingress controller (traefik) to expose the app to the "outside" world.

When all this is complete, you can simply open the application in any browser, on any device in the network by simply poining to [http://newton](http://newton) where *newton* resolves the IP address to your pi. Your pi is now at work.

![](/images/pi/hello-world-3.png)

The entire process is explained beautifully [here](https://www.youtube.com/watch?v=QcC-5fRhsM8) and [written here](https://carpie.net/articles/ingressing-with-k3s). But newer versions of K3s have rendered some parts of the yml inaccurate. 

The updated one can be found [here](https://github.com/devqurious/homecloud/blob/main/yml/hello-world/mysite-nginx.yml) but alas, one day it is sure to be outdated and inaccurate itself. 

```
kubectl create configmap mysite-html --from-file index.html
sudo kubectl apply -f mysite-nginx.yml
```

## Play round

Change the replicas to 3, and boom, you have three pods running now and serving your users. 

```
ubuntu@newton:~$ sudo kubectl get pods -o wide
NAME                            READY   STATUS    RESTARTS   AGE    IP           NODE     NOMINATED NODE   READINESS GATES
mysite-nginx-5559ffd776-xftrv   1/1     Running   0          138m   10.42.0.11   newton   <none>           <none>
mysite-nginx-5559ffd776-64mhj   1/1     Running   0          8s     10.42.0.12   newton   <none>           <none>
mysite-nginx-5559ffd776-vzrh6   1/1     Running   0          8s     10.42.0.13   newton   <none>           <none>
```

Well, right now, it's just serving you at [http://newton/hello-world](http://newton/hello-world).

To modify the message

```
sudo kubectl edit configmap mysite-html
```

To clean up

```
sudo kubectl delete -f mysite.yaml
sudo kubectl delete configmap mysite-hml
```

## An Important Note

Finding a hello world program was surprisingly hard - most of the sample yml files are for apps that are NOT arm-based. So copying them will not work, and you will see the dreaded CrashBackoff status for all the pods. The nginx app in this post works fine. If you're facing problems in deploying an app, check that the app is supported on arm architecture.