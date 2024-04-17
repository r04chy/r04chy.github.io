---
layout: post
title: "Mac Updates Overwriting /etc/ssh_config"
---

As a relatively new Mac user, I frustratingly noticed that following a recent update my /etc/ssh_config had been wiped out.  I connect to a lot of devices via and unfortunately some of them only support older key exchanges, or I have to use specific SSH keys or unusual usernames to connect that I have no control over.

A quick Google resulted in a few people [reporting](https://discussions.apple.com/thread/252554155) the [same](https://arstechnica.com/civis/threads/apple-keeps-messing-up-my-sshd_config.1329195/) issue, so it seems to be normal Apple behaviour, and to be fair, I should have done this the "correct" way when setting up the Mac in the first place.

So... if you're wanting to retain changes to your SSH configuration it's probably worth dumping any specific configurations in unique files in /etc/ssh/ssh_config.d/ which is conveniently included in the /etc/ssh/ssh_config:

![ssh_config.png](/images/Mac/ssh_config.png)

RTFM, etc

