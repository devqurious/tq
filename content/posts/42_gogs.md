---
title: "Part 42: Gogs"
date: 2021-02-06T12:10:51+05:30
thumb_image: "/posts/attachments/gogs_logo.jpg"
omit_header_text: true
draft: false
tags: ["git", "gogs"]
categories: ["HomeCloud"]
---

What services can we move to the homecloud? Well, let's start with github.com. [Gogs](https://gogs.io/) is similar to Github, but is self-hosted.

# Step 1: Create the volume claim

Create a new volume claim that can be used by the gogs-container to store all the persistent data like your repos and configuration. 

Apply the following configuration (remembering to use -n gogs to make sure keep all the gogs related stuff in its own namespace)

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: longhorn-volv-gogs-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 5Gi
```

```
sudo kubectl apply -f longhorn-volv-gogs-pvc.yml -n gogs
```

# Step 2: Create the deployment

Note the name of the image we will pull from docker hub; it is gogs-rpi. Also note that we attach the volume claim we created above to the container. The (longhorn) volume is mounted on /data in the container. The container listens on two ports. Port 22 for SSH based access to git, and port 3000 for HTTP based access to git.  

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gogs-app
  labels:
    name: gogs-app
spec:
  replicas: 1
  selector:
    matchLabels:
      name: gogs-app
  template:
    metadata:
      labels:
        name: gogs-app
    spec:
      containers:
        - name: gogs-app
          image: gogs/gogs-rpi
          ports:
          - containerPort: 3000
            name: gogs-web
          - containerPort: 22
            name: gogs-ssh
          imagePullPolicy: Always
          volumeMounts:
          - name: volv
            mountPath: /data
      volumes:
      - name: volv
        persistentVolumeClaim:
          claimName: longhorn-volv-gogs-pvc
```

# Step 3: Create the service(s)

You need a way to interact with the container from outside world. We will create an abundance of services.

- "gogs" is the service that exposes the HTTP based access to git on port 3000. This is a node port service. This means port 30130 is available on every node in the cluster, and and packets to 30130 are automatically forwarded to port 3000 running inside the gogs-app container. To access, enter http://dtstudios.in:30130

- "gogs-ssh" is the services that exposes the SSH based access to git on port 22. This is also a node port service. The node port is 30131. To access, enter "ssh git@dtstudios.in -p 30131" 

- "gogs-service" is the service that exposes the HTTP based access to port 3000, but this of type "ClusterIP", not node port. This service will be used by Traefik, and allows the user to access the HTTP-based git using a much simpler URL like http://dtstudios.in/gogs (look ma, no ports in the URL)

```
apiVersion: v1
kind: Service
metadata:
  name: gogs
  labels:
    name: gogs
spec:
  ports:
  - port: 3000
    nodePort: 30130
    targetPort: 3000
  selector:
    name: gogs-app
  type: NodePort
---
apiVersion: v1
kind: Service
metadata:
  name: gogs-ssh
  labels:
    name: gogs-ssh
spec:
  ports:
  - port: 22
    nodePort: 30131
    targetPort: 22
  selector:
    name: gogs-app
  type: NodePort
---
apiVersion: v1
kind: Service
metadata:
  name: gogs-service
spec:
  selector:
    name: gogs-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
```

# Step 4: Create the ingress route (optional)

For web based access, we can use path based routing instead of port based routing. This is basically a redirect from port 80 (in the Traefik lod balancer) to port 3000 (inside the gogs-app container)

```
# This ingress point is handled by traefik, and uses path-based routing to
# the gogs service.
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gogs-ingress
  annotations:
    kubernetes.io/ingress.class: "traefik"
    traefik.frontend.rule.type: PathPrefixStrip
spec:
  rules:
  - http:
      paths:
      - path: /gogs
        pathType: Prefix
        backend:
          service:
            name: gogs-service
            port:
              number: 80
```

# Step 5: Verify 

```
sudo kubectl get pvc -n gogs -o wide
NAME                     STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE   VOLUMEMODE
longhorn-volv-gogs-pvc   Bound    pvc-41eccf1e-fb37-4635-8a81-787dc8004ff5   5Gi        RWO            longhorn       9d    Filesystem
```

```
sudo kubectl get pods -n gogs -o wide
NAME                        READY   STATUS    RESTARTS   AGE   IP            NODE      NOMINATED NODE   READINESS GATES
gogs-app-6b88b8779c-v87vw   1/1     Running   0          9h    10.42.2.123   galileo   <none>           <none>
```

```
sudo kubectl get svc -n gogs -o wide
NAME                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE    SELECTOR
gogs                       NodePort    10.43.116.161   <none>        3000:30130/TCP   8d     name=gogs-app
gogs-postgresql            ClusterIP   10.43.220.46    <none>        5432/TCP         7d1h   app=postgresql,release=gogs,role=master
gogs-postgresql-headless   ClusterIP   None            <none>        5432/TCP         7d1h   app=postgresql,release=gogs
gogs-service               ClusterIP   10.43.141.15    <none>        80/TCP           19h    name=gogs-app
gogs-ssh                   NodePort    10.43.69.133    <none>        22:30131/TCP     20h    name=gogs-app

```

```
sudo kubectl get ing -n gogs -o wide
NAME           CLASS    HOSTS   ADDRESS         PORTS   AGE
gogs-ingress   <none>   *       172.16.17.253   80      2d3h
```

# Step 6: Initial configuration

If all the resources are up and running correctly, then it's time to hit the following URL: http://dtstudios.in/gogs, or http://dtstudios.in:30130. This will display the following page. Gogs supports postgres and sqlite, I chose to the latter for simplicity. 

![](/images/pi/gogs.png)

# Step 7: Using Gogs

Finally, the time to reap the rewards. 

First time repo creation...

```
git init
git add .
git commit -m "first commit"
git remote add origin ssh://git@dtstudios.in:30131/devaqurious/homecloud.git
git push -u origin master
```

From then on, use git as usual... adding, committing, and pushing your changes to the repo. All your code gets stored in the in your HA longhorn cluster. 

To view the web interface of gogs, enter https://dtstudios.in/gogs

![](/images/pi/gogs_web.png)

Perfect.O!


# Troubleshooting

```
ssh -vvvvA git@dtstudios.in -p 30131
```

The file that runs on startup is docker/start.sh, which runs a bunch of commands as user "git". To run the web interface manually...

```
EXPORT user=git
./gogs web -p 3000
```

For a clean restart

- delete all contents from the /data directory
- scale to zero (sudo kubectl scale deployment gogs-app --replicas=0 -n gogs)
- scale to one (sudo kubectl scale deployment gogs-app --replicas=1 -n gogs)

First time setup needs to be done via: 

http://dtstudios.in:30130/install (nodeport, not cluster port)

# Other things tried

```
helm repo add incubator https://charts.helm.sh/incubator
"incubator" has been added to your repositories
```

Got the usual dreaded error: 

```
standard_init_linux.go:219: exec user process caused: exec format error
```

Gitlab-ce is another option, but it seemed heavier of the two. 

# References

 - https://www.redhat.com/sysadmin/git-gogs-podman
 - http://dbg.io/local-github-like-source-control-with-gogs-and-docker/
 - https://github.com/zsoltm/docker/blob/gogs-armhf/armhf/apps/gogs/Dockerfile
 - http://dbg.io/local-github-like-source-control-with-gogs-and-docker/
 - https://github.com/kyzdev/gogs-k3s/blob/master/base/gogs-deployment.yaml
 - https://github.com/apk8s/k3s-gitlab



