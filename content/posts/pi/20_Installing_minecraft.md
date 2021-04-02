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
helm install --namespace minecraft --generate-name -f minecraft.yml itzg/minecraft
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




