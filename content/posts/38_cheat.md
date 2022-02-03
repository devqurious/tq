---
title: "Part 38: Kubectl cheat sheet"
date: 2021-01-10T12:10:51+05:30
thumb_image: "/posts/attachments/kubectl.jpg"
omit_header_text: true
draft: false
tags: ["homecloud", "computers"]
categories: ["HomeCloud"]
---

Some nifty little commands.

## Restart all pods belonging to a namespace.

Say you have a bunch of pods that are all part of the "monitoring" namespace, and you need to restart all of them. 

`sudo kubectl -n monitoring rollout restart deploy`

Sometimes this may not work - say one of the pods has a mounted vol. Now, the new pod will not be able to mount the vol as it's already mounted by the old one. In such cases...

```
sudo kubectl get deployments -n rsync
sudo kubectl scale --replicas=0 deployment/hc-rsyncer-app
sudo kubectl scale --replicas=1 deployment/hc-rsyncer-app
```

## Get all the daemon sets

It's like a deployment of pods, except you can set a desired count, and it the daemonset controller will ensure that many pods will run - across all the nodes in the cluster. 

`sudo kubectl get ds -n monitoring`

## Remove a container

Sometimes you forget to create the deployment in its own namespace. The containers run in the default namespace, polluting it. 

```
sudo kubectl get deployment hc-rsyncer-app
sudo kubectl delete deployment hc-rsyncer-app
```
## Something went wrong, what?

Get last set of events...

```
sudo kubectl get events --all-namespaces  --sort-by='.metadata.creationTimestamp'
```

## Delete a pod in a bad state

```
sudo kubectl delete pods [hc-rsyncer-app-54d46f6df5-rxt4s] -n rsync
```

## Delete all resources in a namespace.

```
kubectl delete all --all -n {namespace}
```