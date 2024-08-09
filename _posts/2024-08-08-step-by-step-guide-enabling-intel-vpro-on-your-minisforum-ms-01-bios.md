---
title: "Step-by-Step Guide: Enabling Intel® vPro™ on Your Minisforum MS-01"
author: isaac
date: 2024-08-08 07:00:00 -0700
categories: [Remote Access, IT Administration, Proxmox, Tutorials and Guides]
tags: [Intel® vPro™, Intel® AMT, MS-01, Minisforum, MeshCommander, KVM Over IP]
render_with_liquid: false
toc: true
comments: true
image:
  path: /assets/img/myphotos/vpro/ms-01.png
  alt: 
---

# Configure Minisforum MS-01 BIOS to Enable Intel® AMT/Intel® vPro™

Before diving in, I want to mention that I made several adjustments to the power management settings. While my objective wasn't to achieve the most optimized or power-efficient setup, these changes ensure that the network adapters are consistently available, both for Wake On LAN and AMT/vPro. Your setup requirements may differ. Note: I will not cover Wake On LAN in this post.

## Accessing BIOS

First, press the `Del` key on your keyboard during boot-up. Ensure you have a monitor connected to the device.  
**Note:** I typically use my PiKVM v4, but for some reason, the video wouldn't come up. Using a physical monitor worked perfectly.

Once in the BIOS, select "Setup".

![Setup Screen](/assets/img/myphotos/vpro/20240808102138.png)

Navigate to Advanced, then Onboard Devices Settings.

![Advanced Settings](/assets/img/myphotos/vpro/20240808102243.png)

## Disabling ASPM

Set the Primary Display to `HG`. I've faced inconsistent results when leaving this on `Auto`.

![Primary Display Settings](/assets/img/myphotos/vpro/20240808102345.png)

Select SA-PCIE Port.

![SA-PCIE Port](/assets/img/myphotos/vpro/20240808102531.png)

Ensure your screen resembles the image below. `ASPM` should be disabled at every occurrence. Once done, go back one screen and select PCH-PCIE Port.

![SA-PCIE Port Settings](/assets/img/myphotos/vpro/20240808105238.png)

Again, ensure `ASPM` is disabled everywhere you see it.

![PCH-PCIE Port](/assets/img/myphotos/vpro/20240808105417.png)

![PCH-PCIE Settings](/assets/img/myphotos/vpro/20240808105509.png)

**Note:** Turning off Active-State Power Management (ASPM) may conflict with your requirements. If you need or prefer to leave some of these settings enabled, be aware that one of your devices may become unavailable if the OS determines it is not needed. For the AMT/vPro NIC, this would mean remote rebooting a hung system or turning the machine back on would become impossible.

## Configuring MEBx

Return to the main menu and navigate to MEBx.

![MEBx Menu](/assets/img/myphotos/vpro/20240808105802.png)

![MEBx Password Prompt](/assets/img/myphotos/vpro/20240808132605.png)

When prompted for the current password, type: `admin`.

This next part can be tricky. You must create a complex password with at least one special character like `!` or `$`, some letters, and numbers, including at least one uppercase letter. Enter this complex password twice. Until I figured this out, I encountered several errors that didn't specify a password issue, so don't get tripped up like I did.

Once past the password setting, you'll see a screen similar to the one below. Select `Intel(R) AMT Configuration`.

![Intel AMT Configuration](/assets/img/myphotos/vpro/20240808133018.png)

### Network Setup
Focus on `Network Setup` and `Network Access State`. I'll share screenshots of all my settings, demonstrating a working state that allowed me to use all the features I needed in a headless setup.

![Network Setup](/assets/img/myphotos/vpro/20240808135638.png)

### Redirection Features
Ensure everything on this screen is enabled. All items were already enabled on my MS-01, so I didn't need to change anything.

![Redirection Features](/assets/img/myphotos/vpro/20240808135703.png)

### User Opt-in
I changed the User Opt-in to `NONE` to suit my needs. You can leave this on the default setting if it suits you better.

![User Opt-in](/assets/img/myphotos/vpro/20240808140121.png)

### Network Setup Continued
In the `Network Setup`, select `Intel(R) ME Network Name Settings`.

![Intel ME Network Name Settings](/assets/img/myphotos/vpro/20240808140243.png)

While setting up a FQDN is optional, I recommend it. Setting it as Dedicated ensures no cross-configuration while running Proxmox. Once complete, go back one screen.

![FQDN and Dedicated Option](/assets/img/myphotos/vpro/20240808142156.png)

### TCP/IP Settings
Navigate to `TCP/IP Settings`, then `Wired LAN IPV4 Configuration`. This step is crucial. Configuring a static IP isn't just a best practice; it's the only method that consistently worked for me. I faced mixed results with DHCP. Additionally, a static configuration ensures you always know where to connect, even if your DHCP fails.

Disable DHCP Mode and fill in the relevant details as shown below.

![Static IP Configuration](/assets/img/myphotos/vpro/20240808173209.png)

Go back two screens and change the Network Access State to `Active Network`. Save & Exit the BIOS.

![Network Access State](/assets/img/myphotos/vpro/20240808173859.png)

## Fixing the Black Screen Issue

After completing the steps above, you won't need the monitor plugged in anymore. To ensure you have access to KVM or remote viewing once Proxmox loads, use an HDMI plug, like this one on Amazon: [HDMI Plug on Amazon](https://amzn.to/4dxP99K). It comes with three plugs for around $7 at the time of posting. Without this, you'll see the BIOS, but the screen will go black after anything loads. There are many options available on Amazon, but these are the ones I used and can confirm they work for me.

## Connecting to Intel® AMT/Intel® vPro™ with MeshCommander

Initially, accessing the remote console from my Mac seemed almost impossible—though I might be exaggerating a bit. After going through several unhelpful forums, I finally found a valuable thread on [ServeTheHome](https://forums.servethehome.com/): [Getting vPro Remote KVM Working on Minisforum MS-01](https://forums.servethehome.com/index.php?threads/getting-vpro-remote-kvm-working-on-minisforum-ms-01.43269/). The key advice was to use this Docker image locally: [MeshCmd on Docker Hub](https://hub.docker.com/r/brytonsalisbury/meshcmd).

From my Mac, I deployed the container using [OrbStack](https://orbstack.dev/), which I recently discovered from [Christian Lempa](https://www.youtube.com/@christianlempa) in this [video](https://www.youtube.com/watch?v=aJe7CvQ-aM8). It's a bit overkill for this use case, but it gave me a reason to check out OrbStack.

Launch a web browser and navigate to `http://localhost:3000`. The familiar MeshCommander interface loads, working as expected. Don't forget to select TLS when adding a new computer. Below is a screenshot with my three nodes configured.

![MeshCommander Interface](/assets/img/myphotos/vpro/20240808095130.png)

## Ensuring AMT/vPro Continues to Work with Proxmox

After setting up AMT/vPro for remote access and installing Proxmox, I noticed the Proxmox OS interfered with the NIC adapters running AMT/vPro. The NIC would become inactive after a short period. To resolve this, I blacklisted the driver for the 2.5GB NICs since I only use the 10GB NICs on the MS-01. Your setup may vary, so the following suggestions might not be best for you.

```bash
ip link show
```

![](/assets/img/myphotos/vpro/20240808084307.png)

Take note of the adapters with "enp87" or "enp89"; these are the 2.5GB adapters.

```bash
ethtool -i enp89s0
```

![](/assets/img/myphotos/vpro/20240808085148.png)

Take note of the driver listed; for the MS-01, it should be `igc`.

To stop Proxmox from loading the drivers for these adapters, we need to exempt them from Proxmox OS management.

Create the following file:

```bash
nano /etc/modprobe.d/blacklist-nic.conf
```

Add the content below to the `blacklist-nic.conf` file and save it.

```plaintext
blacklist igc
```

Update initramfs and then reboot.

```bash
update-initramfs -u
reboot
```

After reboot, the network section of Proxmox should look similar to the below.

**Before**

![Before](/assets/img/myphotos/vpro/20240808093422.png)

**After**

![After](/assets/img/myphotos/vpro/20240808093531.png)

**Note:** To disable only a specific NIC, use a udev rule. Below are some commands I would run. Make sure you pick the correct adapter, such as "enp87" or "enp89".

Get detailed information about NICs, including their PCI addresses, using:

```bash
lspci -nn | grep -i ethernet
```

Create or edit a udev rule file in `/etc/udev/rules.d/`, for instance, `70-disable-enp89s0.rules`:

```bash
sudo nano /etc/udev/rules.d/70-disable-enp89s0.rules
```

Add this to the file:

```shell
SUBSYSTEM=="net", ACTION=="add", NAME=="enp89s0", OPTIONS+="ignore_device"
```

Save and exit the editor, then reboot.

## Closing

This guide covers setting up Intel® AMT/Intel® vPro™, or KVM, on your MS-01. It might be a trivial task for those who regularly use Intel® AMT/Intel® vPro™, but it took me longer than expected, and I couldn't find a single comprehensive guide. This post aims to consolidate all the steps into one place. Please let me know your thoughts or if you encounter issues in the comment section below.