---
title: "(How-To) Deploying Cloud-Init Template with Ubuntu on Proxmox with Ansible Playbook - Part 1"
author: isaac
date: 2024-04-08 07:00:00 -0700
categories: [DevOps, Cloud Computing]
tags: [Ubuntu, Proxmox, Ansible, Deployment, VM Templates, DevOps Practices, Cloud-Init Template, Home Lab Automation]
render_with_liquid: false
toc: true
comments: true
image:
  path: /assets/img/myphotos/assembly-line.jpg
---

I wanted to create this walkthrough because, while there are forums and documentation from sources like Proxmox and Ubuntu, I felt that many of the walkthroughs weren't overly comprehensive. Even though my experiences may not mirror yours, and it's quite possible that both cloud-init images and Proxmox will improve their usability over time, I wanted to ensure I had an automated way of deploying my servers. Thus, in part two of this post, I will present an Ansible playbook. But first, I will walk through some of the commands and explain how they work.

"Why create scripts for this?" you may ask. Well, you might want to set up a Docker swarm cluster with three identical Ubuntu instances, create k3s manager/worker nodes, or even configure a templated dual Pi-Hole deployment. Sure, you could manually drop in an image, install packages, or run other Ansible playbooks. Alternatively, you could create and maintain a few versions of these playbooks to rebuild your home lab with just a handful of script executions. That's my preference for sure. I don't know about you, but I tend to tinker and change configurations often, and trying to remember all commands and configurations is just too much for me. Thus, applying DevOps principles and automation to my workflow is how I approach building and managing the systems or applications I host in my home lab.

Longer term I will be creating a blog post about [Semaphore](https://semui.co/) and [Kestra](https://kestra.io/) deploying these templates to my Proxmox cluster. Once I complete the post about Semaphore and Kestra, I will discuss my journey to shifting some solutions based on single Docker hosts, to a k3s cluster, running some of my most basic services in my home lab.

As the title suggests this example will focus on Ubuntu-22.04, however with minimal tweaks, it should work with other Debian-based distributions.

#### Create a VM
Now, I'll start off by showing how to complete some of these tasks using the "qm" command set/create.
Step one is to create a Virtual Machine(VM) that will be used to convert into the template.

```bash
qm create 1234 --name base-ubuntu-template --core 2 --memory 2048 --net0 virtio,bridge=vmbr0
```
- `1234`: This is the ID of the image/VM contained on your Proxmox cluster or host. Note that this ID must be unique across your host or cluster.
- `base-ubuntu-template`: This name can be almost anything and should be based on how you name or organize your templates.
- `2`: This is the number of cores that will be allocated to the processor attached to the VM.
- `2048`: This is the amount of RAM that will be allocated to the virtual machine, with the specification in megabytes.
- `net0`: This is the NIC or network adapter. Assuming you only need one NIC, then it is specified by `net0`. Note that there are still configuration parameters; however, this designates NIC 1 in Proxmox. If you need more than one NIC, you would add an additional `--net1`, along with the necessary configuration.
- `virtio,bridge=vmbr0`: 
  - `virtio`: This is the NIC type. Other options include E1000, etc. If you're unsure why you might need one of the other options, it is recommended to leave it set to `virtio`.
    - `bridge=vmbr0`: This is the type of network connection. A bridge, such as `vmbr0`, connects virtual machines to the physical network using a virtual switch to which the physical NIC of the Proxmox host is attached. It allows virtual machines to appear as individual network nodes on the same LAN as the host. 
    - `vmbr0`: This is typically the default or first NIC on your Proxmox hosts.

#### Download the image
Next up is downloading the cloud image to be used as the base disk image. You can find the release images for Ubuntu 22.04 LTS (Jammy Jellyfish), among others, at the official Ubuntu Cloud Images repository: [https://cloud-images.ubuntu.com/releases](https://cloud-images.ubuntu.com/releases). SSH directly to your Proxmox host, or use the shell available in the web GUI for Proxmox. Use the following commands to navigate to the desired path or choose one of your own preference, and then download the image:

```bash
cd /var/lib/vz/template/iso/
wget https://cloud-images.ubuntu.com/releases/jammy/release/ubuntu-22.04-server-cloudimg-amd64.img
```

#### Import Image
Now import the image to a Proxmox disk
```bash
qm importdisk 1234 /var/lib/vz/template/iso/ubuntu-22.04-server-cloudimg-amd64.img local-lvm
```
- `1234`: This is the ID of the image/VM.
- `/var/lib/vz/template/iso/`: This is the path to the image. Note that it's not absolutely necessary to provide the full path if you have changed directory (`cd`) to the path.
- `ubuntu-22.04-server-cloudimg-amd64.img`: This is the name of the image file. Update this based on the image you have chosen.
- `local-lvm`: This is a default storage location found on most Proxmox hosts, but you may place the image on your NAS or another location. The Ansible script below will allow for changes to this path.

#### Attach Disk
This command takes the disk image you imported and attaches it to the previously created VM.
```bash
qm set 1234 --scsihw virtio-scsi-pci --scsi0 local-lvm:1234/vm-1234-disk-0.qcow2
```
- `1234`: This is the ID of the image/VM.
- `scsihw`: This adds an SCSI connection.
- `virtio-scsi-pci`: This is the SCSI connection type. Note, I don’t suggest you change this unless you need to. For example, you could choose to change it to IDE, but generally speaking, this would only be done in specific circumstances.
- `scsi0`: Used to mark the connection to the SCSI controller. If you were to have more than one, you would change this to `scsi1` as an example.
- `local-lvm`: As mentioned above, this is a default storage location found on most Proxmox hosts, but you may place the disk in any other location available to your Proxmox host.
- `local-lvm:1234/`: The storage location with the folder path which will be the ID of the VM.
- `vm-1234-disk-0.qcow2`: This is the actual disk file.

#### Cloud-Init Configuration
Now add the Cloud-Init drive, set the serial console, boot order, and network to DHCP:
```bash
qm set 1234 --ide2 local-lvm:cloudinit --boot c --bootdisk scsi0 --serial0 socket --vga serial0 --citype nocloud --ipconfig0 ip=dhcp
```
- `1234`: This is the ID of the VM (virtual machine).
- `--ide2 local-lvm:cloudinit`: Attaches a Cloud-Init drive to the VM. The drive contains user data for initializing the VM on the first boot. `local-lvm` is the storage location for the `cloudinit` drive.
- `--boot c`: Sets the VM to boot from the disk (`c` indicates disk boot).
- `--bootdisk scsi0`: Specifies `scsi0` as the boot disk.
- `--serial0 socket`: Adds a serial device (`serial0`) to the VM. This is used, along with the following command, to allow for a virtual console that can be viewed through the Proxmox GUI.
- `--vga serial0`: Sets the display output to use the serial interface. (Note: This option seems incorrect and may have been intended to describe a VGA setting, as `serial0` is more appropriate for use with the `--serial0` option. This may need a correction to something like `--vga std` for VGA output).
- `--citype nocloud`: Specifies the cloud-init configuration format to `nocloud`, indicating that no external cloud service is being used for initialization.
- `--ipconfig0 ip=dhcp`: Configures the network interface `ipconfig0` to retrieve an IP address via DHCP. Cloud-Init will use this configuration to set up the VM's network.

#### Create .pub
For this next step, you need to either create a new file and copy the public key into it or copy the public key to allow for key-based SSH authentication.

Creating it manually:
```bash
nano /var/lib/vz/snippets/sshkey.pub
```
Now manually copy one of your SSH keys. Here is an example of what a key might look like: `ssh-ed25519 AAAB3NzaC1lZDI1NTE5AAAAI5PCrJ1U83BsqkI1MmQ1RSTUvg7j23eSJ8KQXzZmr0 youremail@yourdomain.com`

#### Create Password
Now, if you want to allow for SSH access via username and password, we must hash our password.
```bash
echo -n "YourPasswordHere" | openssl passwd -6 -stdin
```
Here is an example output based on the above: `$6$JRsY9F47HwweJbaT$GA619Vj37xThZSm/7NrLh8DkRkISoei05kxSqF3mzH2I/GhZIB1FdmVE6soqgmt5fZqVPwf8Jls.Hjx14Xits1`
- `"YourPasswordHere"`: Replace this with the desired password, avoiding escape characters, and ensure it is stored in a safe location.
- `-6`: The `-6` option specifies SHA-512 hashing.

> For security reasons, it's recommended to add salting to your commands above, though this is not covered here.
{: .prompt-warning }

#### Cloud-Init Login Configuration
Now we can configure the cloud-init to add the SSH key, username, and password.
```bash
qm set 1234 --sshkeys "/var/lib/vz/snippets/sshkey.pub" --ciuser serveradmin --cipassword "$6$JRsY9F47HwweJbaT$GA619Vj37xThZSm/7NrLh8DkRkISoei05kxSqF3mzH2I/GhZIB1FdmVE6soqgmt5fZqVPwf8Jls.Hjx14Xits1"
```
- `1234`: This is the ID of the VM (virtual machine).
- `"/var/lib/vz/snippets/sshkey.pub"`: The path to the SSH key you created above.
- `serveradmin`: The username you will use to log in and initiate the SSH connection.
- `"$6$JRsY9F47HwweJbaT$GA619Vj37xThZSm/7NrLh8DkRkISoei05kxSqF3mzH2I/GhZIB1FdmVE6soqgmt5fZqVPwf8Jls.Hjx14Xits1"`: Your hashed password that you created above.

#### Increase Disk Size
Expand the size of a VM's disk.
```bash
qm resize 1234 scsi0 +20G
```
- `1234`: This is the ID of the VM (virtual machine).
- `scsi0`: The identifier of the SCSI disk that will be resized. The `0` indicates that it's the first SCSI device attached to the VM. If more SCSI disks are attached, they may be named `scsi1`, `scsi2`, etc.
- `+20G`: The amount by which the disk will be increased. In this case, `20G` means 20 gigabytes. The plus sign (`+`) indicates that the space will be added to the existing disk size.

#### Convert to Template
Convert a VM to a Template in Proxmox
```bash
qm template 1234
```
- `1234`: This is the ID of the VM (virtual machine).

#### Full Script
Now you should have a complete template that you can use to clone as many times as you may need. Below is the full script as explained above.
```bash
qm create 1234 --name base-ubuntu-template --cores 2 --memory 2048 --net0 virtio,bridge=vmbr0

cd /var/lib/vz/template/iso/
wget https://cloud-images.ubuntu.com/releases/jammy/release/ubuntu-22.04-server-cloudimg-amd64.img

qm importdisk 1234 /var/lib/vz/template/iso/ubuntu-22.04-server-cloudimg-amd64.img local-lvm
qm set 1234 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-1234-disk-0
qm set 1234 --ide2 local-lvm:cloudinit --boot c --bootdisk scsi0 --serial0 socket --vga serial0 --citype nocloud --ipconfig0 ip=dhcp
qm set 1234 --sshkeys "/var/lib/vz/snippets/sshkey.pub" --ciuser serveradmin --cipassword "$6$JRsY9F47HwweJbaT$GA619Vj37xThZSm/7NrLh8DkRkISoei05kxSqF3mzH2I/GhZIB1FdmVE6soqgmt5fZqVPwf8Jls.Hjx14Xits1"
qm resize 1234 scsi0 +20G
qm template 1234
```

Stay tuned for the next post where I will convert the above into an Ansible playbook. Let me know how it works for you, or if you have any questions. Please leave a comment or give a like below if you found this helpful.

This is the continuation of this post [“(How-To) Deploying Cloud-Init Template with Ubuntu on Proxmox with Ansible Playbook - Part 2”](https://spaceterran.com/posts/Ansible-Proxmox-Ubuntu-Template-Part-2/)