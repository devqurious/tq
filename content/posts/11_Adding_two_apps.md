---
title: "Part 11: Routing to multiple apps"
date: 2020-12-14T10:10:51+05:30
thumb_image: "/posts/attachments/two.png"
omit_header_text: true
draft: false
tags: ["homecloud", "computers"]
categories: ["HomeCloud"]
---

Now it's simple enough to add a second app - a second container. This time we will install a simple app for the pi that emits details about the container it runs in - [whoami](https://hub.docker.com/r/containous/whoami).

To get both of them going, we need to change the yml files a bit:

[Whoami.yml](https://github.com/devqurious/homecloud/blob/main/yml/whoami.yml)

[Mysite-Nginx.yml](https://github.com/devqurious/homecloud/blob/main/yml/hello-world/mysite-nginx.yml)

As before, deploy them like so

```
sudo kubectl apply -f whoami.yml
sudo kubectl apply -f mysite-nginx.yml
```

Not the differences in the yml specs for the contoller, for the two deployments (both are using the same ingress controller). 

{{< gist devqurious 661cc54ddcd057c6ce859e98c9cc5cd8 >}}


{{< gist devqurious a8fa7629e74279ee3f690bd8440603b9 >}}

Now you have two sites - http://newton/[foo](http://newton/foo) and http://newton/[bar](http://newton/bar).

How cool is that!