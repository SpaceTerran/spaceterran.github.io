---
title: "Netdata Memory Alert on Minisforum MS-01: My Experience and Thoughts"
author: isaac
date: 2025-01-02 07:00:00 -0700
categories: [Troubleshooting, HomeLab]
tags: [Netdata, Performance Monitoring, Minisforum MS-01, Memory Diagnostics, RAM Issues, Homelab Hardware]
render_with_liquid: false
toc: true
comments: true
image:
  path: /assets/img/myphotos/netdata-memory-alert-minisforum-ms01-experience-thoughts/RAM_ERROR.png
  alt: 
---


I’ve recently started testing [**Netdata**](https://www.netdata.cloud/), a real-time performance monitoring and diagnostics tool for systems and applications. So far, I’m impressed with its comprehensive metrics collection and intuitive visualizations.

However, I encountered an alert the other day on one of my Minisforum MS-01 devices:  
`## Warning, System corrupted memory = 0.00391 MB, on <server name>`

![Warning, System corrupted memory](/assets/img/myphotos/netdata-memory-alert-minisforum-ms01-experience-thoughts/email_alert.png)

#### My Setup:

I’m running the **Minisforum MS-01 Mini Workstation**, featuring:

- Intel Core i9-13900H
- 32GB DDR5 RAM
- 1TB SSD
- Dual 10Gbps SFP+, Dual 2.5G RJ45
- USB4, HDMI, PCIe 4.0x16 slot
- Support for multiple M.2 2280/22110/U.2 SSDs

You can check out the exact model here:  
[MINISFORUM MS-01 Mini Workstation](https://amzn.to/404sQDx)

#### Upgrades:

I replaced the original RAM with:

- [Mushkin Redline DDR5 SODIMM - 96GB (2x48GB)](https://amzn.to/3BU1PdN)

And installed two NVMe SSDs:

1. [Samsung 990 PRO Heatsink SSD 4TB (PCIe 4.0)](https://amzn.to/4gQqXBi)
2. [Crucial P3 1TB (PCIe 3.0)](https://amzn.to/3Pn1cwo)

#### My Observations:

Everything has been running flawlessly since May 2024, and I currently have three of these units in my homelab. However, only one of them has triggered this specific Netdata alert. Naturally, I was concerned, so I ran a **MemTest86**, which passed without issues. I haven’t experienced any stability problems before or after the alert in the past 8 months.

![Passing MemTest86](/assets/img/myphotos/netdata-memory-alert-minisforum-ms01-experience-thoughts/pass_memtest.png)

#### Insights and Concerns:

After reading this post on the Netdata community forums: Netdata [1hour_memory_hw_corrupted](https://community.netdata.cloud/t/1hour-memory-hw-corrupted/2104) I decided to share my experience here. It seems this might be a one-off blip rather than the start of a hardware issue??? The affected unit has been rebooted, and I’ve had no further issues for over two weeks.

#### Next Steps:

I haven’t taken the time to reseat the RAM or take additional diagnostic measures yet. If anyone else has encountered a similar issue, I’d love to hear your thoughts and insights in the comments below.