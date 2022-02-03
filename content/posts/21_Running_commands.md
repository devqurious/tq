---
title: "Part 21: Running commands"
date: 2020-12-24T12:10:51+05:30
thumb_image: "/images/pi/exec_command.png"
omit_header_text: true
draft: false
tags: ["homecloud", "computers"]
categories: ["HomeCloud"]
---

The problem with previous post (Part 18) was that that any configurations made by "exec"-ing into the pod are lost if the pod restarts. A viable approach is to use PiHole API's to enable and disable the restricted group. Alas, the group-based API's are not quite there yet. In the future this could be a great option, but right now, we need to look elsewhere. 

There are two approaches. First, create your own version of the PiHole pod which has the necessary cron jobs configured into it. This is definitely the correct way and it will be the subject of discussion in a future post. A second "hacky approach" is that we run a command in the PiHole pod from the host container to enable/disable the group. And we do this at specific times during the day. 

[Here](https://github.com/devqurious/homecloud/blob/main/yml/pihole/ph_groups.py) is script that does exactly that. 

It's a python script, so [pip](https://stackoverflow.com/questions/24137291/ubuntu-pip-not-working-with-python3-4) is needed.

```
curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
sudo python3 get-pip.py
pip --version
```

It also uses the [kubernetes python library](https://github.com/kubernetes-client/python), so install that too. 

```
sudo pip install kubernetes
```

To disable a group named Restricted

```
sudo ./ph_groups Restricted 0
```

To enable it back.

```
./ph_groups Restricted 1
```

Now, like before, [stick this in a cron](https://github.com/devqurious/homecloud/blob/main/yml/pihole/ph_cron.cron), and voila, you have time based domain blocking for specific clients on your network. And even if the PiHole pod restarts your configuration is not persists. Now if the PiHole is restarting around the time when the cron job is about to run, you might miss a beat, but that's quite a corner case and **eventually** the system will do the right thing. 

Now the odd chance of restricted domains being accessible (or not accessible) for sometime...is something that's acceptable.

For now.