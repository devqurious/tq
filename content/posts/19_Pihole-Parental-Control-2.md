---
title: "Part 19: Pihole Parental Control - 2"
date: 2020-12-22T12:10:51+05:30
thumb_image: "/posts/attachments/pi-groups.png"
omit_header_text: true
draft: false
tags: ["homecloud", "computers"]
categories: ["HomeCloud"]
---

Pihole 5 has a feature where you can create a group and assign domain blocklists to just that group only. Now [this is perfect](https://www.vikash.nl/exclude-client-devices-with-pi-hole-5/) when you want to block some domains in your kids computers but don't want to do that for the rest of the machines owned by the adults (yes, this is parental control, our right, as parents! :)). You assign the kids machine to that group, and boom, the kid's machine can no longer access those domains. But there is a problem. 

It's not time-based. 

That means those domains are blocked forever. I don't think I would survive with YouTube being blocked forever on the kid's machine. There would be a general mutiny on board if that happenned. So I had to somehow make this time-based. Allow for sometime during the day, but block the rest of the time. Seems like a [job for cron](https://discourse.pi-hole.net/t/activate-group-with-cron/32660). 

```
# Enable blocking from 3pm to 7pm IST ..

# Enable the blocking (UTC time!)
30 9 * * * sqlite3 /etc/pihole/gravity.db "update 'group' set enabled = 1 where name = 'School'" >> /tmp/pihole_group.log 2>&1
32 9 * * * PATH="$PATH:/usr/sbin:/usr/local/bin/" pihole restartdns >> /tmp/restartpihole.log 2>&1

# Disable the blocking time
58 13 * * * sqlite3 /etc/pihole/gravity.db "update 'group' set enabled = 0 where name = 'School'" >> /tmp/pihole_group.log 2>&1
00 14 * * * PATH="$PATH:/usr/sbin:/usr/local/bin/" pihole restartdns >> /tmp/restartpihole.log 2>&1

# Allowed time from 7:30pm to 9pm IST

# Enable blocking from 9pm to 11am IST
30 15 * * * sqlite3 /etc/pihole/gravity.db "update 'group' set enabled = 1 where name = 'School'" >> /tmp/pihole_group.log 2>&1
2 15 * * * PATH="$PATH:/usr/sbin:/usr/local/bin/" pihole restartdns >> /tmp/restartpihole.log 2>&1

30 5 * * * sqlite3 /etc/pihole/gravity.db "update 'group' set enabled = 0 where name = 'School'" >> /tmp/pihole_group.log 2>&1
32 5 * * * PATH="$PATH:/usr/sbin:/usr/local/bin/" pihole restartdns >> /tmp/restartpihole.log 2>&1
```

We need to now add this cron job to the pihole app. To do that, we need to step inside the pod. 

```
sudo kubectl get pods
```

You can then copy files from the host to the container, using the following command.

```
sudo kubectl cp pihole_cron.cron pihole-7f47c6fc8b-bv4bk:/tmp
```

Here I'm copying a [cron text file](https://github.com/devqurious/homecloud/blob/main/yml/pihole/pihole_cron.cron) into the /tmp directory of the container. To check if it's copied in...

```
sudo kubectl exec --stdin --tty pihole-7f47c6fc8b-bv4bk -- /bin/bash
cd /tmp
```

You should see the cron file in there. Now configure cron.

```
crontab /tmp/pihole_cron.cron
crontab -l (to check)
```

And this will now work perfectly - it will block specific domains for specific clients on the network during specific times of the day. That's spectafularific!

But, alas, all this will work till the pod is restarted, as might happen if the pi is rebooted. The cron jobs will be lost. You will now have to re-run the following commands:

```
sudo kubectl cp pihole_cron.cron pihole-7f47c6fc8b-bv4bk:/tmp
sudo kubectl exec --stdin --tty pihole-7f47c6fc8b-bv4bk -- /bin/bash
crontab /tmp/pihole_cron.cron
crontab -l (to check)
```

We will address this gap soon. But for now, enjoy the time based domain blocking.
