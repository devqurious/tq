---
title: "Part 39: Longhorn Rsync"
date: 2021-02-01T12:10:51+05:30
thumb_image: "/posts/attachments/lsync.jpg"
omit_header_text: true
draft: false
tags: ["backup", "rsync"]
categories: ["HomeCloud"]
---

Finally, the time has arrived to backup computers on your network to the SSD based longhorn volume managed by the cluster. 

```
#!/bin/sh

# Set the path to rsync on the remote server so it runs with sudo.
RSYNC="/usr/bin/rsync"

#Todays date in ISO-8601 format:
TODAY=$(date +"%Y-%m-%d")

#LOG file
LOG="/Users/d1/backup_log.txt"
 
# Source and Destination
SOURCE1="/Users/d1/Documents"
SOURCE2="/Users/d1/Desktop"

DESTINATION="root@172.16.16.16:/data"

rsync -avrzR --exclude=.DS_Store -e 'ssh -p 30037' $SOURCE1 $SOURCE2 $DESTINATION

# Writes a log of successful updates
echo "BACKUP success-$TODAY" >> $LOG

# Clean exit
exit 0

```

Depending on how much you sync, your cluster nodes could spin up for quite a bit. 

![](/images/pi/longhorn_rsync.jpg)

The rsync command has a couple of cool things. For example, the IP address 172.16.16.16 is the IP address of the load balancer, acting as the virtual IP of the cluster. Port 30037 is a node port, which means it exists on every node in the cluster, and that's how the SSH command is able to work. But that's not all. It's the job of Rancher here to make sure that any packets destined for this port, are automatically forwarded to port 22 which is inside the rsync pod, which could be running on any one of the three nodes in the cluster - maybe on Galileo, maybe on Newton. It does not stop there - once the rsync starts, it copies the source directory to mounted volume (/data) which itself is a longhorn volume - and there are two replicas of the data in this volume - all _that_ is managed by Longhorn itself...which it achieves using it's "volume as a service" design (it's containers manage the replication).

Pretty amazing. 

If you now run this script via cron, you now have automated backup of your data - with high availability like never before. If your main source computer gets corrupted, you can still access your data from the SSD drives. If your main source computer goes down AND one of your pi's goes down, you can still get access to your data. If you're main source computer goes down, AND one of your pi's goes down, AND one of your SSD's is corrupted, you can still access your data, as Longhorn will rebuild the data.

If more than this goes down, then...not much to do than to suck it up and move on with life. 

**PS: A note for running this via cron in mac**

Normally, the operation will not be permitted. Install this cron, and verify for yourself.

```
* * * * * cd /Users/foo/Documents/code/homecloud/docker_apps/rsync_container && ./home_sync.sh >> /Users/foo/backp.log 2>&1
```

You have [allow cron](https://apple.stackexchange.com/questions/378553/crontab-operation-not-permitted) in the privacy settings, and sometimes even doing that [might be confusing](https://blog.bejarano.io/fixing-cron-jobs-in-mojave/). 