---
title: "Part 15: Traefik - The Dashboard (1)"
date: 2020-12-18T12:10:51+05:30
thumb_image: "/posts/attachments/traefik.png"
omit_header_text: true
draft: false
tags: ["homecloud", "computers"]
categories: ["HomeCloud"]
---

Traefik is something that's installed by default when you install K3s. It's like the door to your cluster. The external world interacts with the services inside the cluster using Traefik. 

The clients in the external world don't connect directly to the services (/pods) running inside the cluster. Instead they connect to Traefik, which then connects to the services. In computer lingo, this is often called a "reverse proxy." There are many reverse proxies out there - Nginx, HA Proxy and Envoy Proxy - to name a few, but now let's look at Traefik.

Traefik has a dashboard that can be helpful to understand what it does. It's not that straightforward to access the dashboard, so follow carefully. 

First, enable the dasboard.

```
sudo kubectl -n kube-system edit cm traefik
```

This will open a configuration file. Add the important lines as shown below - `[api] dashboard = true`

```
apiVersion: v1
data:
  traefik.toml: |
    # traefik.toml
    logLevel = "info"
    dashboard = "true"
    defaultEntryPoints = ["http","https"]
    [entryPoints]
      [entryPoints.http]
      address = ":80"
      compress = true
      [entryPoints.https]
      address = ":443"
      compress = true
        [entryPoints.https.tls]
          [[entryPoints.https.tls.certificates]]
          CertFile = "/ssl/tls.crt"
          KeyFile = "/ssl/tls.key"
      [entryPoints.prometheus]
      address = ":9100"
    [ping]
    entryPoint = "http"
    [kubernetes]
      [kubernetes.ingressEndpoint]
      publishedService = "kube-system/traefik"
    [traefikLog]
      format = "json"
    [api]
      dashboard = true
    [metrics]
      [metrics.prometheus]
        entryPoint = "prometheus"
kind: ConfigMap
metadata:
  annotations:
    meta.helm.sh/release-name: traefik
    meta.helm.sh/release-namespace: kube-system
  creationTimestamp: "2020-12-20T13:03:59Z"
  labels:
    app: traefik
    app.kubernetes.io/managed-by: Helm
    chart: traefik-1.81.0

```

Restart Traefik

```
sudo kubectl -n kube-system scale deploy traefik --replicas 0
sudo kubectl -n kube-system scale deploy traefik --replicas 1
```

Traefik is now listening but we need to forward the requests to it. Setup port forwarding as shown below:

```
sudo kubectl -n kube-system port-forward deployment/traefik 8080
```

Now test it out - run `curl http://localhost:8080/dashboard/` from within the cluster (the pi) and see that you get a response. The trailing slash is needed.

But how do we see this in our browser, so we can see the dashboard in all it's glory. This is where ssh comes in. 

In the mac machine, i.e. the external client, run the following command:

```
ssh -L 9999:localhost:8080 ubuntu@newton
```

This will basically tunnel all the traffic going to localhost:9999...all the way to localhost:8080 on the remote machine, that's the cluster node in this case (the pi).

Now open a browser and enter `http://localhost:9999/dashboard/` and voila!

![Traefik Dashboard](/images/pi/traefik_dashboard_1.png)

![Traefik Dashboard](/images/pi/traefik_dashboard_2.png)

It's all colorful gobledygook right now. But we will explore further and enter the rabbit hole.

PS: The steps need to be re-done with the node is restarted. (Why? TODO: Make this permanent)