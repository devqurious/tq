---
title: "Part 6: Configure passwordless SSH"
date: 2020-12-09T08:10:51+05:30
thumb_image: "/images/pi/password.jpg"
omit_header_text: true
draft: false
tags: ["homecloud", "computers"]
categories: ["HomeCloud"]
---

We are going to be logging into the master hundreds (maybe thousands?) of times during development. Do we really want to enter the password that many number of times? NO!

Follow the steps [here] (https://www.tecmint.com/ssh-passwordless-login-using-ssh-keygen-in-5-easy-steps/) for a detailed walkthrough but the TL;DR is:

(On the client machine)

```

// create the private and public keys 
ssh-keygen -t rsa

// make the .ssh directory on the server
ssh ubuntu@newton mkdir -p .ssh

// upload the client's public key to the server (master)
cat .ssh/id_rsa.pub | ssh ubuntu@newton 'cat >> .ssh/authorized_keys'

// set permissions
ssh ubuntu@newton 'chmod 700 .ssh; chmod 640 .ssh/authorized_keys'

```

Now if you `ssh ubuntu@newton`, boom, you're in. Look ma, no passwords.

