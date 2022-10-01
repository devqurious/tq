---
title: "Part 34: Environment variables"
date: 2021-01-06T12:10:51+05:30
thumb_image: "/posts/attachments/env.jpg"
omit_header_text: true
draft: false
tags: ["homecloud", "computers"]
categories: ["HomeCloud"]
---

The rsync container now rsync [as a daemon](https://www.jveweb.net/en/archives/2011/01/running-rsync-as-a-daemon.html). This is a daemon that listens on the well known port 873 for incoming connection from other computers utilizing rsync. 

```
/usr/bin/rsync --no-detach --daemon --config /etc/rsyncd.conf rsync_server
```

In the /etc/rsyncd.conf file is a reference to a secrets file that contains the usernames and passwords of all those users who will be allowed to connect to this daemon. How do we get this username/password configuration data into the container? Manually adding it inside the container would not work as it would simply be gone the next time the container came up. 

The way to do is by using environment variables. 

```
env:
    - name: USERNAME
        value: root
    - name: PASSWORD
        value: root
```

Find it below!

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
          env:
           - name: USERNAME
             value: root
           - name: PASSWORD
             value: root
```

When you this container runs, inside the container are two variables whose values you can get by simply using `$USERNAME` and `$PASSWORD`. You're free to use them in scripts too that [you may have copied into the container](/posts/pi/31_adding_rsync/) using Dockerfile. 

