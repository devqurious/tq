---
title: "Part 37: Using SSD"
date: 2021-01-09T12:10:51+05:30
thumb_image: "/posts/attachments/ssd.png"
omit_header_text: true
draft: false
tags: ["homecloud", "computers"]
categories: ["HomeCloud"]
---

Most Pi's run off SDCards. They are fast, cheap but can be unreliable. SDCards were invented to store pictures and videos, not the OS. SSD drives are faster, cheaper and more reliable. And come in huge TB-sized sizes. So why not use them instead.

You need a SSD drive like the one below. 

![](/images/pi/ssd.png)

Then you need a SATA to USB adapter cable like the one below.

![](/images/pi/adapter.png)

Now there are two things to keep in mind when configuring your pi's to boot from SSD drive.

Copy (or shall we say etch) Pi-compatible Ubuntu 20.04 image into the SSD drive using the Balena Etcher program. You can do this from your developmental mac machine. 

Then follow [this legendary blog](/https://jamesachambers.com/raspberry-pi-4-ubuntu-20-04-usb-mass-storage-boot-guide/). 

You will:

1. Update the firmware (if required)
2. Modify the SSD drive using the automatic option.
3. Remove the SD Card. 
4. And reboot.

Voila - your pi will now boot into your Ubuntu OS. Yay.

**Important Note**

Connecting the SSD drive to USB-3 ports on the pi gives you more speed, but in my case it caused horrendous problems - the boot ups were very slow, and often it would not boot up at all. Connecting the SSD drive to USB-2 port solved the problem, but of course, the speeds are compromised too. 

Oh well, hopefully the USB-3 drivers are fixed someday, and we can all enjoy the blazing speeds of a SSD. 

But for now, marvel at the fact that your pi's are now suddenly MUCH more reliable, and can store a whopping amount of data. In my case, the upgrade was from a 64GB SD Card to to a 1000 GB SSD drive. 

Nice!!!