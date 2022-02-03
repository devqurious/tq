---
title: "Part 3: Installing the the OS"
date: 2020-12-06T08:10:51+05:30
thumb_image: "/posts/attachments/Ubuntu.jpg"
omit_header_text: true
draft: false
tags: ["homecloud", "computers"]
categories: ["HomeCloud"]
---

Insert the sdcard into the Macbook Pro. 

- Install the [Balena Etcher](https://www.balena.io/etcher/) program.
- Download the [Ubuntu 20.04 OS image](https://ubuntu.com/download/raspberry-pi) for RPi (depends on model).
- Use the Etcher program to flash the SD card. 
- Connect the RPi to a monitor, keyboard and mouse and network using a *ethernet* cable.
- Once booted, login using the default username and password (ubuntu/ubuntu - you will be asked to change the default password)
- Login with new password, and check the IP address.
```
ip addr show
```
- Disconnect and connect via SSH (SSH server is enabled by default)
```
ssh ubuntu@IP
```
- Update the OS

```
sudo apt-get update
sudo apt-get upgrade
```

Do not disconnect the local keybard, mouse and video yet. Move to the next part to [set a static IP address](/posts/pi/4_pihome_configue_ip)

Set the timezone. 

```
sudo timedatectl set-timezone Asia/Kolkata
```

PS: You dont *have* to connect the pi to keyboard, monitor and mouse. Another option is to simply connect it to the network using an Ethernet cable, and then finding out the DHCP assigned IP address from your router (or whereever DHCP server is running). Then, you can directly SSH in. But having a local keyboard and monitor makes it simpler. 