---
title: "Part 22: Monitoring with cron"
date: 2020-12-25T12:10:51+05:30
thumb_image: "/images/pi/Monitoring_Minecraft.png"
omit_header_text: true
draft: false
tags: ["homecloud", "computers"]
categories: ["HomeCloud"]
---

In the previous part we ran the following command to setup the port forwarding.

```
sudo kubectl port-forward blah blah
```

Alas, the port-forward command is not as stable as our players would like it to be. After all, the port-forwarding is mainly a tool for debugging. Still, there are ways to make it a bit more reliable. We will setup a cron job that will monitor the minecraft server, and if the server is found to be in a problematic state, we will restart the server. 

First the script to check the health of the Minecraft server. This is [the original script](https://raw.githubusercontent.com/vertecx/nagios-plugins/master/check_minecraft.py) that does a great job of monitoring the server, but [the one here](https://github.com/devqurious/homecloud/blob/main/yml/minecraft/check_minecraft.py) is updated to work with python3. 

Run the script like so, and watch the output.

```
check_minecraft.py --hostname 192.168.1.254 -m "Welcome to Minecraft on Home Cloud!" -v
MINECRAFT OK: 0/20 players online
```

The [next script](https://github.com/devqurious/homecloud/blob/main/yml/minecraft/port-forward.py) is a simple one to spawn a new process to port-forward if the above script returns MINECRAFT BAD. 

```
#!/usr/bin/python3

from kubernetes import client, config
import os

# Retrieve the API
config.load_kube_config(config_file="/home/ubuntu/.kube/config")
api = client.CoreV1Api()

pod_list = api.list_namespaced_pod("minecraft")
for pod in pod_list.items:
    print("%s" % pod.metadata.name)
    os.system('sudo kubectl -n minecraft --address 0.0.0.0 port-forward ' + pod.metadata.name + ' 25565:25565')
```

And finally a [bash script](https://github.com/devqurious/homecloud/blob/main/yml/minecraft/spawn_server.sh) to tie them both together.

```
#!/bin/bash
# Script to monitor and restart the mine craft server if necessary.

INSTALL_DIR=/home/ubuntu/install
python3 $INSTALL_DIR/check_minecraft.py --hostname 192.168.1.254 -m 'Welcome to Minecraft on Home Cloud!' >/dev/null 2>&1
ret_code=$?

if [ $ret_code -ne 0 ]; then
    echo 'Minecraft server is not reachable, will try to restart the port-forward'
    if pgrep kubectl; then pkill kubectl; fi
    python3 $INSTALL_DIR/port-forward.py &
else
    echo 'Minecraft server is working fine'
fi

exit 0
```

Now stick this in script in your cron like so.

```
# Check Minecraft Server
* * * * * /home/ubuntu/install/spawn_server.sh & >> /tmp/pihole_group.log 2>&1
```

Now this will check the Minecraft server every minute, and if it finds it unreachable, it will restart it. 

Done.


