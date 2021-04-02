---
title: "Part 30: Hello Docker + K3s"
date: 2021-01-02T12:10:51+05:30
thumb_image: "/images/pi/moby.png"
omit_header_text: true
draft: false
tags: ["homecloud", "computers"]
categories: ["HomeCloud"]
---

(You need to be signed into [Docker hub](https://hub.docker.com/repositories) for some of the commands here)

So far we've been using containers built by other folks. Let's build a simple container for ourselves, and deploy it to our K3s cluster. 

## Install Docker for Mac
The first step is to [install Docker for mac](/https://hub.docker.com/editions/community/docker-ce-desktop-mac/), and then go through the initial tutorials. And [this](/https://stackify.com/docker-build-a-beginners-guide-to-building-docker-images/) is a good one too to build some muscle memory.

## Create the Dockerfile

Then fire up a terminal, create a new directory and cd into it. There, create a new file called `Dockerfile`. 

```
FROM ubuntu:latest
RUN apt-get -y update
RUN apt-get -y upgrade
ENTRYPOINT ["tail", "-f", "/dev/null"]
```

This will Dockerfile will instruct the Docker to create a new image based on Ubuntu, and run an update after the image is built. It will also keep the container in running state with the `tail -f` command running continuously, after the container is created. 

Why is this important? Well, without it, the container would immediately exit after being created. This would be not be taken kindly by K3s, which would try to restart it. The container would start, only to exit again. K3s would try to start it again...and so on. Eventually, K3s would put the container in the dreaded *Crashbackoff* state. Hence that command.  

## Build the container

Our container will run on raspberry pi so it will need to be built for arm. There is an experimental feature in Docker for mac that lets you create a pi-compatible container easily. 

Read [this](https://docs.docker.com/buildx/working-with-buildx/). And [this](https://medium.com/@artur.klauser/building-multi-architecture-docker-images-with-buildx-27d80f7e2408).  

```
docker buildx create --name mybuilder
docker buildx use mybuilder
docker buildx ls 
```

## Now Build the container!

Time to use the builder, and pass the plaform information to the container. `--load` option will load the image into the local docker registry so you can launch a new container from the Docker for mac desktop app.

```
docker buildx build --platform linux/arm64 --tag drathaqm/home_sync --load .
```

The image is now in your local repo. You can now run it from the UI and ensure that it runs correctly.

![](/images/pi/docker_local_repo.png)

Now push to the remote repo. The command is almost the same as the one before. (You can 'Push to Hub' from the UI too)

```
docker buildx build --platform linux/arm64 --tag drathaqm/home_sync --push .
```

Now the container is available in the hub. Let's move to our pi and pull it down from there. 

## Deploy on K3s

Login to the master node, newton. Create a new deployment from some directory. Note the image name is exactly what we used when we pushed it to docker hub (drathaqm/home_sync)

```
ssh ubuntu@newton
vim hc_rsyncer.yml
```

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
```

Now create a new deployment.

```
sudo kubectl apply -f hc_rsyncer.yml
```

After a while, if all goes well...

```
sudo kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
hc-rsyncer-app-7df4c6c65d-bgzcl   1/1     Running   0          56m
mysite-nginx-5559ffd776-nlg69     1/1     Running   3          12d
```

Your very own container, built from scratch (relatively speaking!) is now running in your K3s cluster. Cool!!!

Let's bash in and see the container arch. 

```
sudo kubectl exec -it <pd_name> bash
root@hc-rsyncer-app-7df4c6c65d-bgzcl:/# uname -m
aarch64
```

Arm64 indeed!

## Scale deployment, if you need to

Change the replicas to 2, and boom, two containers running on different nodes. 

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hc-rsyncer-app
  labels:
    name: hc-rsyncer-app
spec:
  replicas: 2
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
```

```
Every 2.0s: sudo kubectl get pods -o wide                                                                                                                          newton: Sat Jan 23 21:06:54 2021

NAME                              READY   STATUS    RESTARTS   AGE    IP            NODE         NOMINATED NODE   READINESS GATES
hc-rsyncer-app-7df4c6c65d-bgzcl   1/1     Running   0          165m   10.42.2.121   copernicus   <none>           <none>
hc-rsyncer-app-7df4c6c65d-nn928   1/1     Running   0          18s    10.42.1.175   galileo      <none>           <none>
mysite-nginx-5559ffd776-nlg69     1/1     Running   3          13d    10.42.1.144   galileo      <none>           <none>


```

## Notes

This container runs two server - nginx AND rsync over SSH. If you were to run the container locally...

```
docker run -it -d  -p 8080:80 -p 2222:22 drathaqm/home_sync
```

...you can access both services thus:

1. Open a browser and type http://localhost:8080
2. Open a terminal and type ssh -p 2222 root@localhost




## Links

Understand the [difference between ENTRYPOINT and CMD](https://www.ctl.io/developers/blog/post/dockerfile-entrypoint-vs-cmd/) and why you may sometimes want to use them BOTH in a Dockerfile.