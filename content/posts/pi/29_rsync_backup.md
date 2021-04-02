---
title: "Part 29: Rsync backup"
date: 2021-01-01T12:10:51+05:30
thumb_image: "/images/pi/rsync_backups.jpg"
omit_header_text: true
draft: false
tags: ["homecloud", "computers"]
categories: ["HomeCloud"]
---

Rsync is a great tool for backing up files and folder. And since I had a spare mac mini lying around (old one, probably the 2012 model) with about 200 gigs of free disk space, it seemed like the perfect little experiment to backup all the files from the frequently used mini to the older one.

## Step 1: Export the dest folder on mac

Follow [this](https://knowledge.autodesk.com/search-result/caas/sfdcarticles/sfdcarticles/Enabling-network-NFS-shares-in-Mac-OS-X.html) guide to configure the destination mac to export a directory via NFS. 

```
sudo vim /etc/exports
/Users/ -mapall=<usrname> -network 172.16.16.0 -mask 255.255.255.0
```

## Step 2: Mount the dest folder on your linux server

Then follow [this](https://behind-the-scenes.net/mounting-a-shared-nas-folder-onto-a-raspberry-pi/) guide to mount the destination directory from one you servers in the cluster. Mount the directory and manually add/delete files to verify that all read/write operations are working correctly. 

## Step 3: Setup passwordless access

You must ensure that there is [passwordless SSH](/posts/pi/6_pihome_configure_passwordless_ssh) access to the source machine else the script cannot run by itself via cron. Passwordless access should be setup for the root user. 

## Step 3: Do the rsync

Here is a simple script that will rsync from source to dest. 

```
#!/bin/sh

TODAY=`date +"%Y%m%d"`
YESTERDAY=`date -d "1 day ago" +"%Y%m%d"`
OLDBACKUP=`date -d "2 days ago" +"%Y%m%d"`

# Set the path to rsync on the remote server so it runs with sudo.
RSYNC="/usr/bin/rsync"

# Set the folderpath on the QNAP
# Dont't forget to mkdir $SHAREPATH
SHAREPATH="/mnt/homecloud/Champ/hc_backup"
 
# This is a list of files to ignore from backups.
# Dont't forget to touch $EXCLUDES
# EXCLUDES="$SHAREPATH/servername.excludes"

#LOG file
# Dont't forget to touch $LOG
LOG="$SHAREPATH/BACKUP_success.log"
 
# Source and Destination
SOURCE="devendrarath@172.16.16.199:/Users/devendrarath/Documents/"

DESTINATION="$SHAREPATH/mini2020"

rsync -avrh --no-perms --no-group --no-owner --info=progress2 $SOURCE $DESTINATION

# Writes a log of successful updates
echo "BACKUP success-$TODAY" >> $LOG

# Clean exit
exit 0
```
## Step 4: Stick the script into cron

This will take a backup every 59 mins. Adjust to suit your taste.  

```
59 * * * * /etc/homecloud/rsync_backup.sh
```

## Note

There is one minor problem with this backup scheme. If newton (the server which does the rsync) goes down, and remains down, the backups will not work. That's not very reliable, as we would expect from a cluster setup. We will fix this in a later part. 

A second problem is that if the destination drive fails, your backup is a goner. Ideally, we want the destination drive to be a RAID 1 (it mirrors the data on two disks) so that even if one drive goes bad, the data is safe on the other one. Or, we can use [Longhorn](https://longhorn.io) which provides for a distributed block storage level backup. We will explore this in a later part too. 

But for now, you can sleep happy knowing that you have a backup in the home cloud. 

