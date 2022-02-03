---
title: "Part 31: Adding rsync to a container"
date: 2021-01-03T12:10:51+05:30
thumb_image: "/posts/attachments/rsync_container.png"
omit_header_text: true
draft: false
tags: ["homecloud", "computers"]
categories: ["HomeCloud"]
---

In [the last post](/posts/pi/30_hello_docker/), we create a plain-vanilla container and ran it in the K3s cluster. Now, let's add some meat to the container, starting with the rsyncd program. 

Update the Dockerfile.

```
FROM ubuntu:latest
ENV LANG C.UTF-8
ENV NOTVISIBLE "in users profile"

RUN apt-get update && \
	apt-get install -y openssh-server rsync && \
	apt-get clean && \
	rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

RUN mkdir /var/run/sshd
RUN sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
RUN sed 's@session\s*required\s*pam_loginuid.so@session optional pam_loginuid.so@g' -i /etc/pam.d/sshd
RUN echo "export VISIBLE=now" >> /etc/profile

COPY entrypoint.sh /entrypoint.sh
RUN chmod 744 /entrypoint.sh

EXPOSE 22
EXPOSE 873

CMD ["rsync_server"]
ENTRYPOINT ["/entrypoint.sh"]
```

The above file depends on the script that is located [here](https://github.com/axiom-data-science/rsync-server/blob/master/entrypoint.sh).

Rebuild the image as before.

```
docker buildx build --platform linux/arm64 --tag drathaqm/home_sync --load .
```

Now when you run the container, you will have rsync available in the container. Enter the container (you can do it from the Docker dashboard UI) and verify.

```
rsync --version
rsync  version 3.1.3  protocol version 31
```

Run `ps -ef` and verify that the rsync daemon is running in the container.

```
ps -ef 
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 16:04 pts/0    00:00:00 /usr/bin/qemu-aarch64 /usr/bin/rsync --no-detach --daemon --config /etc/rsyncd.conf rsync_server
root        41     1  0 16:04 ?        00:00:00 /usr/bin/qemu-aarch64 /usr/sbin/sshd
root        43     0  0 16:04 pts/1    00:00:00 /usr/bin/qemu-aarch64 /bin/sh
root        77    43  0 Jan22 ?        00:00:00 /usr/bin/ps -ef
```

There are two ports open in this container - port 22 and 873. But how do we connect to them? 

PS: This post uses the content from [here](https://github.com/axiom-data-science/rsync-server)
