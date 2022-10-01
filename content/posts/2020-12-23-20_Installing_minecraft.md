---
title: "Part 20: Minecraft"
date: 2020-12-23
thumb_image: "/posts/attachments/minecraft.png"
omit_header_text: true
draft: false
tags: ["ngrok", "minecraft"]
categories: ["HomeCloud"]
---

## Installing

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

At this point your pod will be in "Pending" state. This is because the persistent volume claim is not created yet. Go ahead and create it:

```
sudo kubectl apply -f longhorn-volv-minecraft-pvc.yml -n minecraft
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


![](/posts/attachments/minecraft-server.png)


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

## Updating

If you need to change some of the values in values.yml, do so and then run the following command to update the chart.

```
helm upgrade -n minecraft minecraft itzg/minecraft -f minecraft.yml
```

## Adding persistence

It sucks if you build things and all of it is gone once the pod restarts. To add persistence, follow the steps below. A prerequisite is [longhorn storage](/posts/35_longhorn_storage) where we setup a shared storage for all the pods in the cluster. 

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

## Node affinity

The longhorn volume we used for persistence is a "ReadWriteOnce", which means only one node (newton, copernicus, or galileo) can mount it. If you try to spin up another instance of the minecraft server on another node, then the new instance will fail. 

Try this. Scale it back to 0, and then scale up to 3. Only one instance will come up, the others will be stuck in pending. 

```
sudo kubectl scale --replicas=1 deployment/minecraft-minecraft -n minecraft
```

On restart, if K8s orchestrator brings up the minecraft server on any of the three servers, we will not know which one to connect to. Also, connecting to the external VIP (XG) won't work, as XG will forward request to the cluster, and requests will fail two out of three times. 

The solution is to "stick" the pod to a given node. This can be done quite easily by updating the values for the helm chart. 

#### Step 1: Find node label

```
sudo kubectl get nodes --show-labels
```

The output is an eye chart, but notice the label...

```
newton Ready control-plane,etcd,master 8d v1.20.2+k3s1 beta.kubernetes.io/arch=arm64,beta.kubernetes.io/instance-type=k3s,beta.kubernetes.io/os=linux,k3s.io/hostname=newton,k3s.io/internal-ip=172.16.16.254,kubernetes.io/arch=arm64,kubernetes.io/hostname=newton,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=true,node-role.kubernetes.io/etcd=true,node-role.kubernetes.io/master=true,node.kubernetes.io/instance-type=k3s,type=primary
```

The label `kubernetes.io/hostname=newton` is the one we need.

#### Step 2: Set the node selector

Open up the values XML and add the following:

```
nodeSelector: {"k3s.io/hostname" : "newton"}
```

Run the upgrade commands as shown above.

#### Step 3: Verify

Scale the deployment to 0 and 1, and notice that the new pod always now come up on newton all the time. You can now happily connect to the IP of newton when trying to access the minecraft server.

## Use Nodeport 

Exposing the service as a nodeport will eliminate the need to manually port-forward and greatly improve the reliability of your server. You little clients will thank you for that. Doing this requires that you have understood [this](/posts/25_vip) post which describes how to attach a load balancer in front of your cluster. Assuming you have done all that, it is now easy to configure a NodePort service for your minecraft server. 

Just modify the minecraft.yml to match the following:

```
serviceType: NodePort
  ## Set the port used if the serviceType is NodePort
  nodePort: 30333
  # Set the external port of the service, usefull when using the LoadBalancer service type
  servicePort: 25565
  loadBalancerIP: 172.16.16.16

```

Now you can access the server on the node port of 30333.

      

## Multiplayer for the world

You can extend the minecraft server to anyone on the internet using [ngrok](https://www.ngrok.io). 

  - Create a free acount on ngrok.io
  - Download the arm64 ngrok binary to the newton (the node that runs minecraft server)
  - [Follow the instructions](https://dashboard.ngrok.com/get-started/setup) to set the authtoken.
  - Now run the binary
  
```
./ngrok tcp 30333
```

This will launch a shell with the tunnel information. Now send this information to the Internet person.

![](/posts/attachments/ngrok_hostname.png)

Refer [this link](https://www.ravbug.com/tutorials/mc-ngrok/) for more information on automating this. 