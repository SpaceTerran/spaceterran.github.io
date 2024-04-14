---
title: "Safari can’t connect to iCloud Private Relay"
author: isaac
date: 2024-04-14 07:00:00 -0700
categories: [DNS, Docker, Apple]
tags: [PiHole, dnsmasq, iCloud, Private Relay, Error Fix, iOS]
render_with_liquid: false
toc: true
comments: true
image:
  path: /assets/img/myphotos/safari-cant-connect-to-iCloud-private-relay-error.jpg
  alt: Safari can’t connect to iCloud Private Relay
---

Do you run a Pi-Hole or another ad blocker? Are your iOS devices screaming "Private Relay is temporarily unavailable due to a technical problem" or "Safari can’t connect to iCloud Private Relay"? I recently had my spouse mention that they were experiencing a lot more pages not loading. Initially, I thought it might just be Google ad links or something else that should be blocked due to our Pi-Hole ad lists. However, after taking a closer look, I was presented with these messages.

 
![Safari can’t connect to iCloud Private Relay](/assets/img/myphotos/safari-cant-connect-to-iCloud-private-relay.png){: .w-50 .left}
![Private Relay is temporarily unavailable due to a technical problem](/assets/img/myphotos/private-relay-is-temporarily-unavailable.png){: .w-50 .right}
\
\
\
\
\
\
\
\
\
Before I discuss the fix, let's talk about what iCloud Private Relay is. Apple provides a fairly detailed description of the feature, available to users who purchase additional iCloud storage (e.g., 50GB: $0.99 or 200GB: $2.99 plan). You can learn more about it here: [About iCloud Private Relay](https://support.apple.com/en-us/HT212614) (Published Date: August 31, 2023). Essentially, Apple creates a kind of VPN that proxies all your internet traffic through its service in the name of "privacy." Privacy can be a nebulous term, but the oversimplified explanation is that using your phone's DNS to resolve website addresses, along with data like cookies, device name, location, etc., can potentially be sent to various websites you visit or apps you use.

The idea behind Apple's "privacy" concept is that by aggregating all these data points (e.g., from many iPhone users), it becomes much harder to isolate and track a specific individual for targeted ads or analytics. While there is some benefit to having your phone traffic run through a VPN, such as when using public Wi-Fi, there is still a question of how much you can trust Apple with your data. VPN services that prioritize privacy typically emphasize non-logging policies, but Apple's terms may not be as strict as privacy advocates would like. What's your take on this? Does it concern you, or are you considering turning it off? Share your thoughts in the comments.

If you just want to disable iCloud Private Relay, you can consult Apple's guide and find the "Turn iCloud Private Relay off" section: [Protect your web browsing with iCloud Private Relay on iPhone](https://support.apple.com/en-sg/guide/iphone/iph499d287c2/ios). Conversely, if you have guest users on your Wi-Fi, you can improve their experience by tweaking a setting on your Pi-hole. This setting allows you to block iCloud Private Relay in such a way that users can simply acknowledge that it won't work on your network by clicking a button, thus avoiding random page load failures illustrated in the screenshots above.

Apple offers guidance for network administrators on how to manage iCloud Private Relay traffic: [Prepare your network or web server for iCloud Private Relay](https://developer.apple.com/support/prepare-your-network-for-icloud-private-relay). By setting your DNS server to respond with a `NXDOMAIN` error for mask.icloud.com and mask-h2.icloud.com URLs, iOS devices are informed that iCloud Private Relay cannot connect, stopping persistent retry attempts.

You may or may not know that Pi-hole's default blocking method can be customized according to your needs, but it currently ships with the default set to `NULL`. The `NULL` response blocks queries by answering with the "unspecified address" (0.0.0.0 or ::). There is considerable debate about whether `NULL` is the best way to ensure that ad blockers are most effective. However, my research suggests that when a `NXDOMAIN` is sent, developers may have their hardware or applications either attempt another method (e.g., direct IP) or, if the software is not sophisticated enough, it may repeatedly query your DNS server, leading to increased load as it tries to send home its telemetry data. So, unless you have a specific need, I have personally found `NULL` to be the most effective, and I wouldn't suggest changing the default settings within Pi-hole.

Since Apple's feature is designed to detect when it receives a `NXDOMAIN` response, you need to configure your Pi-hole to send a `NXDOMAIN` for these specific DNS names. For all other responses, it will continue operating normally with `NULL`. Pi-hole uses dnsmasq behind the scenes, so you simply need to create a file in the directory `/etc/dnsmasq.d` on your Pi-hole instance or Docker container. While this post won’t cover all the possible locations or instructions to create a Docker-based Pi-hole, if you're using a default installation, placing the file below in this directory and then restarting Pi-hole will ensure that a `NXDOMAIN` is sent to your iOS devices while maintaining the standard function of Pi-hole, which is `NULL`.

```bash
cd /etc/dnsmasq.d
sudo nano 10-mask-icloud.conf
```

Paste the following lines into the file you just created:

```text
server=/mask.icloud.com/
server=/mask-h2.icloud.com/
```

After saving the file, make sure to restart your Pi-hole to apply the changes. Once done, users should receive a prompt like the one below when they connect to your network, allowing them to choose "Use Without Private Relay." This action will prevent the message from reappearing in the future, and users should no longer face unexpected page loading issues.

![Use Without Private Relay](/assets/img/myphotos/use-without-private-relay.png){: .w-50}

I'd love to hear if this workaround helped you or if you've tackled the issue differently. Feel free to share in the comments below.