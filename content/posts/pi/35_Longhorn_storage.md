---
title: "Part 35: Longhorn storage"
date: 2021-01-07T12:10:51+05:30
thumb_image: "/images/pi/longhorn_dashboard.png"
omit_header_text: true
draft: false
tags: ["homecloud", "computers"]
categories: ["HomeCloud"]
---

Now we have a rsync container up and running in our K3s cluster, and it can listen for rsync requests from clients, and on receiving them it will immediately store the files in the container. That's great. 

Until you restart the container, or the node on which the container runs. Then, poof, all the data is gone. To fix this problem, we need to use a persistent storage volume. 

As always, there are options. But by far, the easiest one that will also ensure that our data is never lost is the one using [Longhorn](https://github.com/longhorn/longhorn) storage. 

So let's install it. 

```
mkdir longhorn
cd longhorn/
git clone https://github.com/longhorn/longhorn
helm repo add longhorn https://charts.longhorn.io
helm repo update 
sudo kubectl create namespace longhorn-system
helm install longhorn ./longhorn/chart/ --namespace longhorn-system
sudo kubectl get pods -n longhorn-system
```

Once all the pods are up, setup port forwarding. Here we're forwarding all traffic on 8002 to longhorn svc listening on port 80 (which is a nice dashboard....wait for it...)

On the newton node (can be any node)
```
sudo kubectl -n longhorn-system --address 0.0.0.0 port-forward svc/longhorn-frontend 8002:80
```

Now, on the mac (i.e. a machine outside the cluster)

```
ssh -L 9999:localhost:8002 ubuntu@newton
```

Finally, enjoy the dashboard at http://localhost:9999/#/dashboard.

## Data corruption

Due to an un-graceful shutdown, or other reasons, your volume can get corrupted. When this happens, the container that depends on this volume will fail to start. Viewing the logs for that container, will show an error like so:

```
Warning  FailedMount  26m (x3 over 42m)     kubelet  Unable to attach or mount volumes: unmounted volumes=[config], unattached volumes=[default-token-pzfww config custom-dnsmasq]: timed out waiting for the condition
/dev/longhorn/pvc-dcaa9f25-03e1-432d-af6d-c3b85820e0e6: UNEXPECTED INCONSISTENCY; RUN fsck MANUALLY.
```

Is'nt that beautiful - an error, and a solution side by side. So log into the node where the volume exists, and run the following command (change the vol locations to whatever matches for your setup.

```
sudo fsck -cvf /dev/longhorn/pvc-dcaa9f25-03e1-432d-af6d-c3b85820e0e6 <- You can get this path from the longhorn dashboard.
```

And in no time, your container should be up and running. 

## Important note

Installing longhorn causes two default storage classes to be created, as mentioned here: https://github.com/civo/kube100/issues/12. 

```
sudo kubectl get storageclass -o wide
NAME                   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  4d2h
longhorn (default)     driver.longhorn.io      Delete          Immediate              true                   3d11h
```

This is wierd, and may cause problems when installing apps, such as pihole. 

```
helm install --version '1.8.22' --namespace pihole --values ph_values.yml pihole mojo2600/pihole
Error: persistentvolumeclaims "pihole" is forbidden: Internal error occurred: 2 default StorageClasses were found
```

The fix is in the link above ...

```
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

## Volume corruption

A longhorn volume can get corrupted, like any other type of volume. This usually happens due to an unclean shutdown. If this happens, then the pods that are using that pvc volume will fail to start and the container will be stuck in "ContainerCreating." To find out which volume is corrupted, run the `describe pod` command which will tell you the name of the volume. 

Then, from the longhorn UI find out which node is attached to the volume and ssh into that node. Then run the fsck command (withtout the -a option) to manually fix all the errors. (Press a to accept all as yes)

For example:

```
sudo fsck /dev/longhorn/pvc-dcaa9f25-03e1-432d-af6d-c3b85820e0e6
```

Now restart the container, and the container should come up just fine!




