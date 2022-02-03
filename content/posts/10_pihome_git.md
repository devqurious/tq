---
title: "Part 10: GitHub and VSCode Rsync"
date: 2020-12-13T09:10:51+05:30
thumb_image: "/images/pi/sync.png"
omit_header_text: true
draft: false
tags: ["homecloud", "computers"]
categories: ["HomeCloud"]
---

We've started writing enough code now to neccesitate git. Login to GitHub, create a new project repository and follow the instructions to connect it to a repo in your development box.

It's great that you can add all your code to the repo, but copying the code over on every change to the pi is a pain. This is where the [vscode rsync extension](https://github.com/thisboyiscrazy/vscode-rsync#workspaces) comes in handy. Once this is all setup, then as soon as you hit save, it will copy over any changes to the pi. Sweet!

Click on the exploding box icon, search for rsync and install it.

![](/images/pi/rsync-1.png)

Next, click on Code | Preferences | Settings (on mac) and scroll to rsync settings section.

![](/images/pi/rsync-2.png)

Then configure away.

![](/images/pi/rsync-3.png)

![](/images/pi/rsync-4.png)

![](/images/pi/rsync-5.png)

![](/images/pi/rsync-6.png)

Make some change to your local folder and watch it seamlessly rsync to the server...on every save.
