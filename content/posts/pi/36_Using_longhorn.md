---
title: "Part 36: Using longhorn"
date: 2021-01-08T12:10:51+05:30
thumb_image: "/images/pi/longhorn_dashboard.png"
omit_header_text: true
draft: false
tags: ["homecloud", "computers"]
categories: ["HomeCloud"]
---

Now that we have setup [Longhorn](/posts/pi/35_longhorn_storage/) it's time to [put it to use](https://longhorn.io/docs/0.8.1/volumes-and-nodes/create-volumes/). Here's a couple of things to keep in mind.

- It's the job of the IT administrator that's managing the cluster to create the persistent volume (pv). So the first step is to create the persistent volume of the appropriate size. 

- It's the job of the developer now to submit a persistent volume claim when building the application.

You can execute the first part from the dashboard using the "Create Volume" button, or if you want, use a manifest. Note the replica count - 

For the second part, the manifest is below. This is a request for 20GB of persistent storage for the rsync app. (You can create this from the dashboard too!)

```
vim longhorn-volv-rsync-pvc.yml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: longhorn-volv-rsync-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 20Gi
```

To use this persistent volume claim, update the container...(see the last part)

```
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
          volumeMounts:
          - name: volv
            mountPath: /data
          env:
           - name: USERNAME
             value: root
           - name: PASSWORD
             value: root
      volumes:
      - name: volv
        persistentVolumeClaim:
          claimName: longhorn-volv-rsync-pvc
```

