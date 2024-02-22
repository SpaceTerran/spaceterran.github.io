---
title: "Ansible Playbook for Deploying Docker Swarm and GlusterFS"
author: isaac
date: 2024-02-17 02:00:00 -0700
categories: [Ansible, Ansible Playbook, Docker, Docker Swarm]
tags: [Docker Swarm, GlusterFS, Ansible, Proxmox, Terraform, Ubuntu Server, Virtual Machine Provisioning, Deployment Guide, High Availability, James Turland, Jim's Garage, Techno Tim]
render_with_liquid: false
toc: true
comments: true
image:
  path: /assets/img/myphotos/Ansible-Docker-Swarm-GlusterFS.jpg
  alt: Ansible Playbook for Deploying Docker Swarm and GlusterFS
---

Hello there! Today, I'm going to walk through an Ansible playbook, allowing the deployment of Docker Swarm and GlusterFS. Now, I must share that the inspiration for this playbook came from a YouTuber by the name of James Turland of Jim's Garage. I would highly recommend you check out his YouTube page: [Jim's Garage YouTube](https://www.youtube.com/@Jims-Garage) or check out the repository where the initial inspiration came from: [Jim's Garage GitHub](https://github.com/JamesTurland/JimsGarage/blob/main/Docker-Swarm). And if you would rather follow along with Jim's script, check out the video reviewing using his script: [Use Docker Swarm! Auto Deploy Script with Highly Available Storage - GlusterFS](https://www.youtube.com/watch?v=Ws68qHWIFMM).

## What is Docker Swarm and GlusterFS, anyway?
Docker Swarm and GlusterFS are powerful tools that, when combined, offer a robust solution for managing containerized applications with high availability and scalable storage. Docker Swarm simplifies the process of managing multiple Docker hosts, allowing them to work together as a single, virtual Docker host. It provides native clustering capabilities, easy scaling of applications, and ensures that your services are always running. On the other hand, GlusterFS is an open-source, distributed file system capable of scaling to several petabytes and handling thousands of clients. GlusterFS clusters together storage bricks to provide a single, unified storage volume that is highly available and scalable.

By integrating Docker Swarm with GlusterFS, you can create a resilient environment where your applications can scale seamlessly and your data is stored across multiple nodes, ensuring redundancy and fault tolerance. This setup is ideal for businesses looking to deploy critical applications that require high availability and data integrity.

## Lets Get Started
With all of that out of the way, the first thing you need to do is create some virtual machines. Of course, if you are just trying to demonstrate this script, you can start off with 3-5 virtual machines running on the same host. But, as I'm sure some of the sharp folks out there will say, why run a Docker Swarm on a single hardware host as this defeats the purpose. And yes, 100% I agree, that is the case. However, it's always a good idea to test things out before you go ahead and use them in your own environment, so you do have that option if you don't have multiple physical hosts.

Now, for my example below, I will walk you through how I have it set up in my lab on three different Proxmox hosts.

Here are the high-level steps:
1. Create virtual machines on my Proxmox hosts using templates and Terraform.
2. Update an Ansible hosts file.
3. Run the Ansible playbook, called `swarm.yml`.

## Creating the Proxmox templates
Now if you don't already have some proxmox templates, I highly recommend you take a look at Jim's video on how to create a cloud-init image on Proxmox here: [Jim's Proxmox Cloud-Init Video](https://www.youtube.com/watch?v=Kv6-_--y5CM) _*Note: This was part of his multi-part Kubernetes video series, but this video only covers creating the template_. Or 
[Techno Tim's - Perfect Proxmox Template with Cloud Image and Cloud Init](https://www.youtube.com/watch?v=shiIi38cJe4). These two videos will surely get you started if you don't have some templates already. 

>As for the cloud image, I use "Ubuntu Server 22.04 LTS (Jammy Jellyfish)" found here: [Ubuntu Jammy Server Cloud Image](https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img).
{: .prompt-info }

>Warning: I have had issues with "QCow2 UEFI/GPT Bootable disk image with Linux-KVM KVM optimised kernel" image, so I would suggest you stick with the one I shared above. And as always, check for the latest guidance from Canonical to ensure you are using a safe-to-use image.
{: .prompt-warning }

## Create Virtual Machines using terraform
Now, I'm going to create 2 servers per host to be used with this demonstration. Below is a small export of my Terraform script that I use to provision the Proxmox virtual machines. In a later post, I will cover this more in detail, but you could use the below to get you started. I'm using the Terraform provider: [bpg/terraform-provider-proxmox](https://registry.terraform.io/providers/bpg/proxmox/latest/docs).

```terraform
## terraform variables ###

variable "vm_count_int_swarm_primary" {
  description = "Number of VMs to create"
  default     = 2
}

variable "vm_name_base_int_swarm_primary" {
  description = "Base name of the VMs to be created"
  default     = "proxmox-host-1"
}

variable "ipaddr_base_int_swarm_primary" {
  description = "Base IP address to assign to the VMs"
  default     = "10.10.10.1/29"
}
```
Now, you can use the above to duplicate these resources 3 times. For example, create additional resources `vm_count_int_swarm_primary` -> `vm_count_int_swarm_secondary`, `vm_name_base_int_swarm_primary` -> `vm_name_base_int_swarm_secondary`, and so on.

```terraform
### terraform resources ###
resource "proxmox_virtual_environment_vm" "vm_int_swarm_primary" {
  provider  = proxmox
  count     = var.vm_count_int_swarm_primary
  name      = "${var.vm_name_base_int_swarm_primary}-${count.index}"
  node_name = "proxmox-node1" # Update to your own Proxmox node name.
  vm_id     = "20${count.index}"
  clone {
    node_name = "proxmox-node1" # Update to your own Proxmox node name.
    vm_id     = 5002 # replace with your template's VM ID
    full      = true
  }
  on_boot = false
  started = false
  operating_system {
    type = "l26"
  }
  agent {
    enabled = true
  }
  memory {
    dedicated = 4096
    floating  = 4096
    shared    = 4096
  }
  cpu {
    architecture = "x86_64"
    cores        = 2
    sockets      = 1
    type         = "host"
    units        = 100
  }
  network_device {
    bridge = "vmbr0"
    model  = "virtio"
  }
  disk {
    datastore_id = "local-lvm"
    interface    = "scsi0"
    size         = 30
    discard      = "on"
    ssd          = true
  }
  lifecycle {
    ignore_changes = [
      started,
      agent,
    ]

  }
}
```
Now, for the above, you're going to want to take a look at values `node_name`, `vm_id`, and `datastore_id`, as these will need to be updated with your environment's variables. And change any other relevant items specific to your virtual machine CPU, RAM, etc.

>Please note: this Terraform script is not complete and very rough, to demonstrate how you can get up and running quickly. In the future, I will look to parameterize more of the Terraform code as an example.
{: .prompt-info }

## Creating an Ansible Hosts(inventory) file
Now that I have my virtual machines, I'm going to set up my Ansible hosts file. Below is just an example, so be sure to update this based on your environment. The Ansible playbook is looking for at least 3 sections: `int_swarm`, `int_swarm_managers`, `int_swarm_workers`. As always, feel free to make changes to naming conventions, etc., to best suit your needs. But if you decide to change these names, you will also need to update them in the `swarm.yml` playbook.

```yaml
# Group for all hosts in the infrastructure
int_swarm:
  hosts:
    manager-1:
      ansible_host: 10.10.10.1
      ansible_ssh_user: admin_user
    worker-1:
      ansible_host: 10.10.10.2
      ansible_ssh_user: admin_user
    manager-2:
      ansible_host: 10.10.10.3
      ansible_ssh_user: admin_user
    worker-2:
      ansible_host: 10.10.10.4
      ansible_ssh_user: admin_user
    manager-3:
      ansible_host: 10.10.10.5
      ansible_ssh_user: admin_user
    worker-3:
      ansible_host: 10.10.10.6
      ansible_ssh_user: admin_user

# Group for manager nodes within the swarm
int_swarm_managers:
  hosts:
    # Only include the manager nodes
    manager-1:
      ansible_host: 10.10.10.1
      ansible_ssh_user: admin_user
    manager-2:
      ansible_host: 10.10.10.3
      ansible_ssh_user: admin_user
    manager-3:
      ansible_host: 10.10.10.5
      ansible_ssh_user: admin_user

# Group for worker nodes within the swarm
int_swarm_workers:
  hosts:
    # Only include the worker nodes
    worker-1:
      ansible_host: 10.10.10.2
      ansible_ssh_user: admin_user
    worker-2:
      ansible_host: 10.10.10.4
      ansible_ssh_user: admin_user
    worker-3:
      ansible_host: 10.10.10.6
      ansible_ssh_user: admin_user
```
## Ansible playbook - swarm.yml
Below is the actual Ansible playbook. Please ensure you read through each step and understand the actions it's going to take. Copy the script contents here to the filename of `swarm.yml` or anything else that works best for you. Then copy the file to the administrative server or a machine on your network that is set up to connect to these virtual machines we just created.

>Warning: Please take into consideration the latest industry-standard IT security best practices before running something like the below in a production environment, but this should be more than enough to get you started.
{: .prompt-warning }

```yaml
- name: Install Docker and GlusterFS dependencies
  hosts: int_swarm  # Targets the 'int_swarm' group of hosts
  become: true  # Elevates privileges
  tasks:
    - name: Install Docker and GlusterFS  # Descriptive name for a block of tasks
      block:  # Groups tasks together
        - name: Install required packages  # Installs dependencies for Docker and GlusterFS
          ansible.builtin.apt:  # Uses the apt module for package management
            name:  # Lists packages to be installed
              - ca-certificates
              - curl
              - gnupg
              - software-properties-common
              - glusterfs-server
            state: present  # Ensures packages are installed
            update_cache: true  # Updates the package cache first

    - name: Download Docker's official GPG key  # Adds Docker's GPG key for package verification
      ansible.builtin.get_url:
        url: https://download.docker.com/linux/ubuntu/gpg
        dest: /etc/apt/keyrings/docker.asc
        mode: '0644'  # Sets file permissions
        force: false  # Avoids re-downloading if the file already exists

    - name: Retrieve dpkg architecture  # Gets the system's architecture
      ansible.builtin.command: dpkg --print-architecture
      register: dpkg_arch_result  # Saves command output for later use
      changed_when: false  # Marks the task as not changing the system

    - name: Set dpkg architecture fact  # Saves the architecture as a fact for later use
      ansible.builtin.set_fact:
        dpkg_arch: "{{ dpkg_arch_result.stdout }}"

    - name: Retrieve Ubuntu version codename dynamically  # Identifies the Ubuntu version
      ansible.builtin.shell: |
        set -o pipefail && grep 'VERSION_CODENAME=' /etc/os-release | cut -d'=' -f2
      args:
        executable: /bin/bash
      register: codename_result
      changed_when: false

    - name: Set version codename fact dynamically  # Saves the Ubuntu version codename as a fact
      ansible.builtin.set_fact:
        version_codename: "{{ codename_result.stdout }}"

    - name: Add Docker to sources list  # Configures apt to use Docker's repository
      ansible.builtin.lineinfile:
        path: /etc/apt/sources.list.d/docker.list
        line: "deb [arch={{ dpkg_arch }} signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu {{ version_codename }} stable"
        create: true
        state: present
        mode: '0644'

    - name: Update apt-get  # Refreshes the package database
      ansible.builtin.apt:
        update_cache: true

    - name: Install Docker and plugins  # Installs Docker and related packages
      ansible.builtin.apt:
        name:
          - docker-ce
          - docker-ce-cli
          - containerd.io
          - docker-buildx-plugin
          - docker-compose-plugin
          - python3-docker
          - python3-pip
        state: present

    - name: Install Docker Compose using pip
      ansible.builtin.pip:
        name: docker-compose, jsondiff
        state: latest

    - name: Start and enable GlusterFS service  # Ensures the GlusterFS service is running
      ansible.builtin.service:
        name: glusterd
        state: started
        enabled: true

    - name: Ensure GlusterFS brick directories exist  # Creates directory for GlusterFS volume
      ansible.builtin.file:
        path: "/gluster/volume1"
        state: directory
        mode: '0755'
        owner: root
        group: root

- name: Initialize Docker Swarm on first manager
  hosts: int_swarm_managers[0]  # Targets the first manager in the 'int_swarm_managers' group
  become: true
  tasks:
    - name: Check Docker Swarm status  # Checks if the host is part of a Swarm
      ansible.builtin.shell: docker info --format '{{ "{{.Swarm.LocalNodeState}}" }}'
      register: docker_swarm_status
      changed_when: false

    - name: Initialize Docker Swarm  # Initializes the Swarm if not already active
      ansible.builtin.shell:
        cmd: docker swarm init --advertise-addr {{ hostvars[inventory_hostname]['ansible_default_ipv4']['address'] }}
      when: "'inactive' in docker_swarm_status.stdout"  # Conditional execution
      register: swarm_init
      changed_when: "'Swarm initialized' in swarm_init.stdout"

    - name: Retrieve Docker Swarm manager token  # Gets token for joining as a manager
      ansible.builtin.shell: docker swarm join-token manager -q
      register: manager_token
      changed_when: false

    - name: Retrieve Docker Swarm worker token  # Gets token for joining as a worker
      ansible.builtin.shell: docker swarm join-token worker -q
      register: worker_token
      changed_when: false

- name: Join remaining managers to Docker Swarm
  hosts: int_swarm_managers:!int_swarm_managers[0]
  become: true
  tasks:
    - name: Check Docker Swarm status before attempting to join
      ansible.builtin.shell: docker info --format '{{ "{{.Swarm.LocalNodeState}}" }}'
      register: docker_swarm_status
      changed_when: false

    - name: Join Swarm as manager
      ansible.builtin.shell:
        cmd: docker swarm join --token {{ hostvars[groups['int_swarm_managers'][0]]['manager_token'].stdout }} {{ hostvars[groups['int_swarm_managers'][0]]['ansible_default_ipv4']['address'] }}:2377
      when: hostvars[groups['int_swarm_managers'][0]]['manager_token'].stdout is defined and docker_swarm_status.stdout != "active"
      register: swarm_join
      changed_when: "'This node joined a swarm as a manager' in swarm_join.stdout"

    - name: Label Docker Swarm manager nodes  # Applies a label to manager nodes for identification
      ansible.builtin.shell:
        cmd: docker node update --label-add manager=true {{ item }}
      loop: "{{ groups['int_swarm_managers'] }}"
      loop_control:
        loop_var: item
      when: swarm_join is changed
      changed_when: false

- name: Join workers to Docker Swarm
  hosts: int_swarm_workers  # Targets the worker nodes in the Swarm
  become: true
  tasks:
    - name: Check if node is part of a swarm  # Verifies if the node is already in a Swarm
      ansible.builtin.shell: docker info --format '{{ "{{.Swarm.LocalNodeState}}" }}'
      register: swarm_state
      changed_when: false

    - name: Join Swarm as worker if not already part of a swarm  # Joins the Swarm as a worker
      ansible.builtin.shell:
        cmd: docker swarm join --token {{ hostvars[groups['int_swarm_managers'][0]]['worker_token'].stdout }} {{ hostvars[groups['int_swarm_managers'][0]]['ansible_default_ipv4']['address'] }}:2377
      when: swarm_state.stdout != 'active'
      register: swarm_join
      changed_when: "'This node joined a swarm as a worker' in swarm_join.stdout"

- name: Configure GlusterFS on first manager
  hosts: int_swarm_managers[0]  # Again targets the first manager for GlusterFS configuration
  become: true
  tasks:
    - name: Check if GlusterFS volume staging-gfs exists  # Checks for the existence of a GlusterFS volume
      ansible.builtin.shell: gluster volume info staging-gfs
      register: volume_info
      ignore_errors: true
      changed_when: false

    - name: Probe GlusterFS peers and create volume  # Probes peers and creates a GlusterFS volume if not existing
      block:
        - name: Probe peers for GlusterFS
          ansible.builtin.shell: gluster peer probe {{ item }}
          loop: "{{ groups['int_swarm'] }}"
          when: volume_info.rc != 0
          register: peer_probe
          changed_when: "'peer probe: success' in peer_probe.stdout"

        - name: Create GlusterFS volume
          ansible.builtin.shell:
            cmd: >
              gluster volume create staging-gfs replica {{ groups['int_swarm'] | length }}
              {% for host in groups['int_swarm'] %}
              {{ hostvars[host]['ansible_default_ipv4']['address'] }}:/gluster/volume1
              {% endfor %}
              force
          when: volume_info.rc != 0
          register: volume_create
          changed_when: "'volume create: success' in volume_create.stdout"

        - name: Start GlusterFS volume
          ansible.builtin.shell: gluster volume start staging-gfs
          when: volume_info.rc != 0
          register: volume_start
          changed_when: "'volume start: success' in volume_start.stdout"

- name: Mount GlusterFS on all Swarm nodes
  hosts: int_swarm_managers, int_swarm_workers  # Targets both managers and workers for GlusterFS mount
  become: true
  gather_facts: true
  tasks:
    - name: Ensure GlusterFS volume mounts on boot  # Configures fstab for the GlusterFS volume
      ansible.builtin.lineinfile:
        path: /etc/fstab
        regexp: '^localhost:/staging-gfs\s+/mnt\s+glusterfs'
        line: 'localhost:/staging-gfs /mnt glusterfs defaults,_netdev 0 0'
        create: true
        mode: '0644'

    - name: Mount GlusterFS volume immediately  # Mounts the GlusterFS volume
      ansible.builtin.mount:
        path: /mnt
        src: 'localhost:/staging-gfs'
        fstype: glusterfs
        opts: defaults,_netdev
        state: mounted

    - name: Adjust permissions and ownership for GlusterFS mount  # Sets proper permissions for the mount
      ansible.builtin.file:
        path: /mnt
        owner: root
        group: docker
        state: directory
        recurse: true

```
## Running the playbook
Now, run the command in the directory where your hosts and `swarm.yml` have been created/copied.

```ansible-playbook swarm.yml ```

You can add ```--ask-become-pass``` to the end of the previous command if you need to pass a sudo password. 

Additionally, if you don't have an `ansible.cfg` file that contains your Hosts (inventory) file, add ```-i /path/to/hosts/file/hosts```.

The guide includes detailed instructions on creating Proxmox templates, provisioning virtual machines with Terraform, setting up the Ansible hosts file to categorize servers into managers and workers, and the actual Ansible playbook. This playbook covers installing Docker and GlusterFS dependencies, initializing Docker Swarm, joining nodes to the swarm, configuring GlusterFS on the first manager, and ensuring GlusterFS mounts on all Swarm nodes.

>I'd love to hear what you think about the script or your questions if you run into issues. Please see the comments section below.
{: .prompt-tip }