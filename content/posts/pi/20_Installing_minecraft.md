---
title: "Part 20: Minecraft"
date: 2020-12-23T12:10:51+05:30
thumb_image: "/images/pi/minecraft.png"
omit_header_text: true
draft: false
tags: ["homecloud", "computers"]
categories: ["HomeCloud"]
---

Adding the world's most popular video game to the k3s cluster seems like a good idea. There are a couple of posts that guide us in this endeavour - [this](https://www.jeffgeerling.com/blog/2020/raspberry-pi-cluster-episode-4-minecraft-pi-hole-grafana-and-more ) and [this](https://github.com/itzg/minecraft-server-charts/tree/master/charts/minecraft). 

And [this](https://github.com/itzg/minecraft-server-charts/blob/master/charts/minecraft/values.yaml) and [this](https://github.com/itzg/minecraft-server-charts)

Prepare the repo.

```
helm repo add itzg https://itzg.github.io/minecraft-server-charts/
helm repo update
helm search repo itzg
```

Install all the resources in it's own namespace. 

```
sudo kubectl create namespace minecraft
```

Configure the server. Create a new file called [minecraft.yml](https://github.com/devqurious/homecloud/blob/main/yml/minecraft/minecraft.yml) and place in the default values. If you want to have a safe environment, set the difficulty to peaceful (*not* easy as there are zombies in the easy mode too), and gameMode to creative. 

```
helm install --namespace minecraft minecraft -f minecraft.yml itzg/minecraft
```

Once it starts running, find the pod name and tail the logs...

```
sudo kubectl logs -f [podname] -n minecraft
```

If all goes well, the minecraft server is up and running. Check the port on which the Minecraft service is listening on:

```
sudo kubectl get service -n minecraft
```

The default is 25565. Setup a port forward so that you can access it from outside the cluster. 

```
sudo kubectl -n minecraft --address 0.0.0.0 port-forward [podname] 25565:25565
```

0.0.0.0 is important. If you follow the instructions and choose 127.0.0.1 it will not work. The former means that the port forwarding will work on all interfaces, not just the localhost interface. And for sure, you're minecraft client is not running on the pi itself. It will be running outside the cluster.

Now if you `telnet [IPAddressOfPi]:25565` you should see a response. You're now done with the server stuff. 

Download the [Minecraft launcher](https://www.minecraft.net/en-us). Yes, you need to sign in using a Microsoft account. Yes, you need to pay 26 bucks to play the game. Yes, if you intend to play multiplayer, you have to pay 26 bucks for each player. Once you have paid it, you can see the following screen (when you click on Multiplayer - this is NOT visible when you're in demo mode). 

![minecraft-server](/images/pi/minecraft-server.png)

Login, and play!

At this point the K3s cluster is running PiHole and Minecraft. And it will literally be running hot - when I checked the temperature in the PiHole admin UI it was 64 degrees! But to be fair to the Pi 4B, it handled the game smooth as butter. 

Utterly butterly delicious it was!

PS: A couple of other commands that may be handy.

If you want to just upgrade the minecraft with new values:

```
helm upgrade -f minecraft.yml minecraft-1609161211 itzg/minecraft -n minecraft
```

Check the helm charts installed:

```
helm ls -n minecraft
```

## Updating the values

If you need to change some of the values in values.yml, do so and then run the following command to update the chart.

```
helm upgrade -f minecraft.yml minecraft itzg/minecraft -n minecraft
```

## Adding persistence

It sucks if you build things and all of it is gone once the pod restarts. To add persistence, follow the steps below. A prerequisite is [this post](/posts/pi/35_Longhorn_storage.md/) where we setup a shared storage for all the pods in the cluster. 

### Create the persistent volume claim

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: longhorn-volv-minecraft-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 1Gi
```

```
sudo kubectl apply -f longhorn-volv-minecraft-pvc.yml -n minecraft
```

### Update minecraft.yml

```
persistence:
  annotations: {}
  ## minecraft data Persistent Volume Storage Class
  ## If defined, storageClassName: <storageClass>
  ## If set to "-", storageClassName: "", which disables dynamic provisioning
  ## If undefined (the default) or set to null, no storageClassName spec is
  ##   set, choosing the default provisioner.  (gp2 on AWS, standard on
  ##   GKE, AWS & OpenStack)
  ##
  storageClass: "longhorn"
  dataDir:
    # Set this to false if you don't care to persist state between restarts.
    enabled: true
    existingClaim: "longhorn-volv-minecraft-pvc"
    Size: 1Gi

```

### Run the upgrade command

```
helm upgrade --namespace minecraft minecraft -f minecraft.yml itzg/minecraft
```

And voila - your minecraft now has persistence!

### Use Nodeport 

Exposing the service as a nodeport will eliminate the need to manually port-forward and greatly improve the reliability of your server. You little clients will thank you for that. Doing this requires that you have understood [this](posts/pi/25_vip/) post which describes how to attach a load balancer in front of your cluster. Assuming you have done all that, it is now easy to configure a NodePort service for your minecraft server. 

Just modify the minecraft.yml to match the following:

```
serviceType: NodePort
  ## Set the port used if the serviceType is NodePort
  nodePort: 30333
  # Set the external port of the service, usefull when using the LoadBalancer service type
  servicePort: 25565
  loadBalancerIP: 172.16.16.16

```

### Multiplayer for the world

You can extend the minecraft server to anyone on the internet using [ngrok](www.ngrok.io). First, create a free account. Then [follow the instructions](https://dashboard.ngrok.com/get-started/setup) to run the ngrok binary on any machine outside the cluster. 