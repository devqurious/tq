---
title: "Part 44: Cron Script to Cluster"
date: 2022-10-05
thumb_image: "/posts/attachments/scc.jpg"
omit_header_text: true
draft: false
tags: ["docker", ""]
categories: ["HomeCloud"]
---

I needed a container that would could continuously monitor things - websites, other apps, the weather, servers. The monitoring was going to be done in a python script. This post explains how you can go from a script that can monitor something to deploying a pod in K3s so that script can run continuously till the end of time. 

#### Test the script(s)

Here is the script that we want to run periodically. It tries to login to site. All the data it needs is read from environment variables. 

```
import argparse
import requests
import base64
import json
import datetime

from ping3 import ping, verbose_ping
import os
from zoneinfo import ZoneInfo
from urllib.parse import urlparse

d_username = os.environ['DEVUSERNAME']
d_url = os.environ['DEVURL']
q_username = os.environ['QAUSERNAME']
q_url = os.environ['QAURL']

password = os.environ['PASSWORD']
token_name = os.environ['TOKEN']

def env(url):
	domain = urlparse(url).netloc
	domain = domain.split('.')
	env = domain[0] + domain[1]
return env

  
# Returns None, or token upon successfull login.
def login(user_name, password, url):

	sample_string = user_name + ":" + password
	sample_string_bytes = sample_string.encode("ascii")
	base64_bytes = base64.b64encode(sample_string_bytes)
	base64_string = base64_bytes.decode("ascii")

	headers = {
	'Content-Type': "application/json",
	'Authorization': token_name + " " + base64_string
	}

	now = datetime.datetime.now(tz=ZoneInfo('Asia/Kolkata'))
	now_format_string = now.strftime("%d-%B:%H")
	log_header_line = now_format_string + "," + env(url)

  
	#Exit if network is down
	if(ping('google.com') == False):
	print(log_header_line + ", Network is DOWN!")
	return None

  

	#Login test
	
	response = requests.request("GET", url, headers=headers, timeout = 30)
	if response.status_code == 201:
		print(log_header_line + ", Logged in successfully" )
		data = json.loads(response.text)
		api_url = data['apis']['upe']['ng_url'][:-3]
		csrf = data['csrf']
		token_value = data['token']
		return token_value
	else:
	
		print(log_header_line + ", Unable to login due to error code: ", response.status_code)
		return None

return None

  
login(d_username, password, d_url)
```

Here is the script that reads the environment variables and dumps it into a file. This will be needed by cron. 

```
#!/bin/bash

# dump environment variables into a file so we can read in cron - ha!
printenv | sed 's/^\(.*\)$/export \1/g' >> /etc/profile.d/env.sh
```

Finally, here is a cron script that cron will run at fixed intervals

```
# must be ended with a new line "LF" (Unix) and not "CRLF" (Windows)
* * * * * echo "4...Waiting..." >> /var/log/cron.log 2>&1
*/5 * * * * . /etc/profile.d/env.sh && /usr/local/bin/python3 /root/central-check-noargs.py >> /data/cron.log 2>&1
# An empty line is required at the end of this file for a valid cron file.
```

As you can see, this script sends its output /data/cron.log. That does not exist yet, but it will be created during deployment.

#### Create the Dockerfile
Now that we have the scripts and we have tested they work, it's time to place them in a container. That is achieved by creating a Dockerfile and running the appropriate docker command. Make sure you have Docker desktop running.

```
FROM python:3

  

#Install Cron

RUN apt-get update

RUN apt-get -y install cron

  

#Install deps

RUN pip install requests

RUN pip install ping3

  

# Copy hello-cron file to the cron.d directory

COPY cron-job /etc/cron.d/cron-job

  

# Init script

COPY init_env.sh /root/init_env.sh

RUN chmod 744 /root/init_env.sh

RUN /root/init_env.sh

  

# Give execution rights on the cron job

RUN chmod 0644 /etc/cron.d/cron-job

  

# Apply cron job

RUN crontab /etc/cron.d/cron-job

  

# Create the log file to be able to run tail

RUN touch /var/log/cron.log

  

# Add the script to the Docker Image

ADD central-check-noargs.py /root/central-check-noargs.py

  

# Give execution rights on the cron scripts

RUN chmod 0644 /root/central-check-noargs.py

  

# Run the command on container startup

CMD cron && tail -f /var/log/cron.log
```

This Dockerfile basically copies the scripts to the /root folder and then runs cron, and begins tailing the log file so that the container does not exit immediately. Inside the cron script is the command to run the actual script periodically.

#### Build and test on local

##### Build
```
docker buildx build --platform linux/arm64 --tag drathaqm/central-check-noargs --load .
```

##### Run 
```
docker run -d drathaqm/central-check-noargs
```

##### List
```
docker ps
```

##### Login to shell
```
docker exec -it [33678e799c78] bash
```

Once you have logged in, check the log files to make sure cron is working correctly. 

#### Push image 
Once you have verified the container is working correctly on Docker Desktop, it's time to push the image to Docker Hub so it can be pulled down from there by the homecloud cluster.

```
docker buildx build --platform linux/arm64 --tag drathaqm/central-check-noargs --push .
```

#### Deploy the container

Create the deployment manifest file and then apply it from within the cluster. The maifest file should contain the name of the pod, the volume mounts, and environment variables. 

```
sudo kubectl apply -f central-check-deploy.yml -n monitoring
```

### Test in the cluster

Now, the container is running on the K3s cluster. Check that it is running, and then copy the running container name.
```
watch sudo kubectl get pods -n monitoring
```

Exec into the container.
```
sudo kubectl exec -it [central-check-app-7f8df7c7f8-swdqj] -n monitoring -- /bin/sh
```

Check the /var/log/cron.log to ensure the script is running and doing its job.
```
tail -f /data/cron.log
```

Note you can skip 11, and 12 if you just want to check the logs.

```
sudo kubectl logs -f central-check-app-7f8df7c7f8-5jjrc -n monitoring
```

### Rinse and repeat

These are some shortcuts that be used to cut down the number of keystrokes. 

##### During build  
```
docker exec -it $(docker run -d $(docker build -q .)) bash
```

##### During deployment

Step 1: On the development machine.
```
buildx build --platform linux/arm64 --tag drathaqm/central-check-noargs --push .
```

Step 2: In the cluster.

First pull the image. Note that a `rollout restart` option will actually pull the latest images as that's what's been specified in the deployment manifest file. 
```
sudo kubectl rollout restart deployment/central-check-app -n monitoring
```

Next, run and exec (in a single command)
```
sudo kubectl exec -it $(sudo kubectl get pods -n monitoring -o=name |  sed "s/^.\{4\}//" | grep "central") -n monitoring -- bash
```

Refer here for above [command](https://stackoverflow.com/questions/35797906/kubernetes-list-all-running-pods-name). 


### Additional commands:

To stop the container:
```
sudo kubectl scale --replicas=0 deployment/central-check-app -n monitoring
```

To run the container. This command will pull a new image from hub.docker.com if it exists.
```
sudo kubectl scale --replicas=1 deployment/central-check-app -n monitoring
```

Copy file from the container to the host
```
sudo kubectl cp monitoring/central-check-app-844658957d-q6cgk:/data/cron.log cron.log
```

### References

[Passing commands to scripts](https://galea.medium.com/docker-runtime-arguments-604593479f45)
[CMD vs ENTRYPOINT](https://www.bmc.com/blogs/docker-cmd-vs-entrypoint/)
https://medium.com/@muneeburrehman2610/kubernetes-persistent-volume-for-beginners-a13cbe5bdeea
