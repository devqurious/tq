---
title: "Part 12: CrashBackOff!"
date: 2020-12-15T11:10:51+05:30
thumb_image: "/images/pi/Error.png"
omit_header_text: true
draft: false
tags: ["homecloud", "computers"]
categories: ["HomeCloud"]
---

Sometimes the app you download will not start.  It's status will initially be CrashBackOff, and then eventually it will settle into ERROR state.

```
ubuntu@newton:~/homecloud/yml$ sudo kubectl get pods
NAME                                        READY   STATUS    RESTARTS   AGE
whoami-deployment-9db9fcdc4-r4h4c           1/1     Running   0          12h
mysite-nginx-5559ffd776-sjs9l               1/1     Running   0          5h5m
rpi-container-deployment-7445cfcd9d-5g5n6   0/1     Error     3          65s
```

`Kubectl logs` is helpful to know why it all went south.

```
ubuntu@newton:~/homecloud/yml$ sudo kubectl logs rpi-container-deployment-7445cfcd9d-5g5n6
2020/12/13 12:16:38 Unix socket /var/run/docker.sock does not exist
```

Check [this excellent article](/https://managedkube.com/kubernetes/pod/failure/crashloopbackoff/k8sbot/troubleshooting/2019/02/12/pod-failure-crashloopbackoff.html) out when you run into this case. And you will run into it. 