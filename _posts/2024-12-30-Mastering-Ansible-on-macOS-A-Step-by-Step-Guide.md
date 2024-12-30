---
title: "Mastering Ansible on macOS A Step by Step Guide"
author: isaac
date: 2024-12-30 07:00:00 -0700
categories: [Automation, DevOps, System Administration, macOS, Ubuntu, HomeLab, Tutorials and Guides]
tags: [Ansible, Homebrew, SSH Authentication, Mac Setup, Ubuntu Server, Terminal Commands, Playbook, SSH Key Generation, Proxmox, Step-by-Step Guide]
render_with_liquid: false
toc: true
comments: true
image:
  path: /assets/img/myphotos/Mastering-Ansible-on-macOS-A-Step-by-Step-Guide/Mastering-Ansible-on-macOS-A-Step-by-Step-Guide.png
  alt: 
---

Today, I'm going to show you how to install Ansible on a Mac, and prepare a target Ubuntu server with SSH authentication, which will make using Ansible easier in the future. There are many ways to install Ansible. Here's the [official link to Ansible](https://docs.ansible.com/ansible/latest/installation_guide/installation_distros.html) for installing the software on various systems.

There are plenty of ways to install software on a Mac; however, I use Homebrew (Brew).

## Let's Dive into the World of Brew!

Homebrew, affectionately called Brew, is a fantastic package manager for macOS and Linux. It lets you easily install, update, and manage software packages right from the command line. Just think of it as the App Store, but for all the geeky tools and utilities you can imagine, streamlined into terminal commands. It's super handy because it resolves dependencies and organizes your software environment with just a few simple commands.

### Installing Brew

Copy the code below into your Mac terminal, then hit the "enter/return" key:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

You will be asked to continue by hitting the "enter/return" key again. Then it will install the software on your computer after some time. Once this is completed, you will be presented with some text. Follow it along or use the commands below. **NOTE:** These are the commands for my terminal window. Please note the "spaceterran" username, and you will need to update this based on your username (the account you have signed into the Mac with).

```bash
echo >> /Users/spaceterran/.zprofile
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> /Users/spaceterran/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"
```

Once that is completed, check to see if you have Brew up and running by running the command `brew help`.

### Installing Ansible

Now that Brew is installed, let's install Ansible. Are you ready?! In the same window, enter the command below into the terminal window:

```bash
brew install ansible
```

### Setting Up the Target Machine

For this demonstration, I have a small home lab. I suspect you do too, and are looking for ways to help automate or script the configuration of your system(s). I won't cover setting up the VM; however, I'm using a Proxmox host, and I downloaded the Ubuntu Server 24.04.1 LTS [here](https://releases.ubuntu.com/24.04.1/ubuntu-24.04.1-live-server-amd64.iso), and installed Ubuntu server as normal.

The next steps are based on a fresh installation. I haven't made any changes to the Ubuntu instance, so if you are in the same state as I am, these commands should work for you.

### Preparing the Target with SSH Key-Based Authentication

Before we proceed, let me explain what I mean by preparing our target. I want to enable SSH Key-Based authentication so that when I SSH into the server, I'm not prompted for a password. This is important to assist us in the automation process when using Ansible. While you can pass an additional parameter for the server's password, this may not be sustainable if you have multiple servers to manage and run your playbooks on. Having a bunch of passwords lying around isn't a great idea either.

### Creating an SSH Key

With all that said, let's create an SSH key on your Mac for use when connecting to our new Ubuntu server. Note, the next two commands should work on most Linux distributions as well—e.g., Ubuntu, Debian, or Windows WSL.

Before running the command below, change `your_email@changethis.com`. Yes, you could leave this, but it isn't recommended, and you will likely accumulate multiple keys over time. It would be helpful to have some sort of identifier.

```bash
ssh-keygen -t ed25519 -C "your_email@changethis.com"
```

Once you hit enter, you'll be asked where the public and private keys should be stored. I recommend the default location, so simply hit the "enter/return" key to continue.

Next, you'll be asked for a passphrase. Remember, we are using this key for automation, so you have two options: set a key and pass that key in each time you run the playbook, or hit "enter/return" to continue without a password. In my future posts, this is how I have my SSH key authentication set up, so if you want to follow along in the future, I suggest this.

**IMPORTANT:** This is not appropriate for all scenarios; if your keys are stolen, a bad actor will be able to log into your machines without a password. So just like with passwords in source control, **DO NOT** save these in source control (e.g., GitHub).

You'll be asked to confirm the passphrase. Again, just hit "enter/return". Once complete, you'll be at the normal prompt again. It will output some random art and the path to your public and private keys.

### Installing the SSH Key on Ubuntu

Now let's install this new key onto our Ubuntu server. In the same window, enter the following:

_Remember to update the_ `spaceterran` _(username entered when installing ubuntu) and_ `192.168.52.5` _(IP address of your ubuntu server) with your specific environment details._

```bash
ssh-copy-id spaceterran@192.168.52.5
```

The screen will explain what keys it will install and ask you to continue. Type `yes`, then hit "enter/return".

It confirms the key that it will install is not already installed, and asks you to enter the SSH password of the user you created when installing Ubuntu. Enter it now, then hit "enter/return".

If you have completed the steps correctly, you will see a message stating the keys that were installed and a suggestion to try logging in. Let's do this now

*Remember to update the `spaceterran` and `192.168.52.5` with your specific environment details.*

```bash
ssh spaceterran@192.168.52.5
```

### Running the First Playbook

We are now ready to run our first playbook. Let's try something easy. Below is a very basic Ansible playbook that runs the terminal command for checking the disk space. Let's create a file in the same terminal window:

```bash
nano check_disk_space.yml
```

Now copy the following and save your file:

```yaml
---
- name: Check free disk space on specified Ubuntu server
  hosts: all
  gather_facts: no
  tasks:
    - name: Gather disk space information
      command: df -h /
      register: disk_space
    - name: Display disk space
      debug:
        msg: "{{ disk_space.stdout }}"
```

To run this playbook, use:

```bash
ansible-playbook -i <your_host>, -u <your_username> check_disk_space.yml
```

or, in my case:

```bash
ansible-playbook -i 192.168.52.5, -u spaceterran check_disk_space.yml
```

![screenshot playbook ran](/assets/img/myphotos/Mastering-Ansible-on-macOS-A-Step-by-Step-Guide/screenshot-playbook-ran.png)

If everything was run properly, you should see the disk space of your Ubuntu machine.

That's all I'm going to cover now. However, this is a primer for future Ansible playbooks that we'll create and use. Let me know if this helped you, or if there's something I missed that you would like to ask about in the comments section below.
