---
title: "Part 23: Safe search"
date: 2020-12-26T12:10:51+05:30
thumb_image: "/posts/attachments/safe_search.png"
omit_header_text: true
draft: false
tags: ["homecloud", "computers"]
categories: ["HomeCloud"]
---

Google search is great, and it would be hard to get by without it. But it's also a dangerous thing in the hands of a minor - the wrong search, or a typo can expose the kid to all kinds of nonsense. With PiHole you can force every search to be a safe search and thereby reduce the risk greatly. You don't need to configure this on every device on your network - just do it once in PiHole. 

To force safe search create a CNAME entry as below: 

![](/images/pi/safe-search.png)

Try searching something, you should see safe search automatically enabled now.

![](/images/pi/safe.png)


Now if you were to search something inappropriate (deliberately, or not!) you're shown this:

![](/images/pi/blocked.png)

It can be circumvented. Only if your 10 year old knows DNS and how to configure it. That would be impressive, and scary at the same time. 