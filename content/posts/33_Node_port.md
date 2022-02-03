---
title: "Part 33: Exposing a service using NodePort"
date: 2021-01-05T12:10:51+05:30
thumb_image: "/posts/attachments/nodeport.jpg"
omit_header_text: true
draft: false
tags: ["homecloud", "computers"]
categories: ["HomeCloud"]
---

The way to overcome the proble of the last part is to expose the ports in the rsync container (port 22) is to use the NodePort service from K3s. This is more complex, as it does now require an additional entity - a service object.


```
apiVersion: v1
kind: Service
metadata:
  name: hc-rsyncer-service
spec:
  # Expose the service on a static port on each node
  # so that we can access the service from outside the cluster
  type: NodePort

  # When the node receives a request on the static node (30037)
  # it will forward to all pods with the label 'name' set to 'hc-rsyncer-app'
  selector:
    name: hc-rsyncer-app
  ports:
    # Three types of ports for a service
    # port - port exposed internally in the cluster
    # nodePort - static port assigned on each of the nodes
    # targetPort - the container port to send the requests to.
    - port: 22
      nodePort: 30037
      targetPort: 22
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hc-rsyncer-app
  labels:
    # This key/value is used by the NodePort service to direct the requests
    # to this container (name = hc-rsyncer-app)
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

Apply this manifest now, and you should be able to hit the service from outside the cluster. Of course, you will need to port forward 30037 in the load balancer to the cluster. But once it hits the cluster, K3s will ensure the traffic will be routed to the correct node, depending on where the container is running. 

```
nc -vz 172.16.16.16 30037
Connection to 172.16.16.16 port 30037 [tcp/*] succeeded!
```


PS: Thank you [Matthew Palmer](https://matthewpalmer.net/kubernetes-app-developer/articles/service-kubernetes-example-tutorial.html). Your explanation was better than the official documentation IMHO.