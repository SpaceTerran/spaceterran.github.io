---
title: Error's Enccountered While Installing Docker on Ubuntu 22.04 using Ansaible + Portainer
author: isaac
date: 2024-01-26 18:00:00 +0800
categories: [Ansible, Docker, DevOps, Python, Portainer, IT Automation]
tags: [Error Fix, Ansible Playbook, Docker, Docker-compose, Jinja2 Templating, Ubuntu, Secure Repository, GPG Error, Key Not Available, gpg, asc]
render_with_liquid: false
toc: true
comments: true
---
Here's an error message you might encounter:

> Failed to update apt cache: Updating from such a repository can't be done securely and is therefore disabled by default. See apt-secure(8) manpage for repository creation and user configuration details., GPG error: https://download.docker.com/linux/ubuntu jammy InRelease: The following signatures couldn't be verified because the public key is not available: NO_PUBKEY 7EA0A9C3F273FCD8, The repository 'https://download.docker.com/linux/ubuntu jammy InRelease' is not signed.
{: .prompt-danger }

[Jump right to the fix](#addressing-the-error-the-following-signatures-couldnt-be-verified-because-the-public-key-is-not-available-no_pubkey-7ea0a9c3f273fcd8)

My intention was to install Docker and Python components to use Ansible to deploy docker-compose files on my Ubuntu host and then install Portainer. I started off with installing the docker playbook `install_docker.yml`, then I ran `install_portainer.yml`. The error I encountered was with `install_docker.yml`, however, I would like to share a bit about the Ansible file type `.yml.j2` and the Jinja2 templating engine.

With Jinja2, you can dynamically customize Docker Compose files based on variables that wouldn't traditionally be defined within them. It offers an added layer of flexibility and customization. Think about scenarios where you need to deploy containers with varying configurations, ports, volumes, or service options. Jinja2 allows this variety without creating separate individual files for each. Instead, variables can be defined and altered at runtime or passed from Ansible playbooks, streamlining the creation and deployment of Docker environments dynamically.

## Dynamically Templating docker-compose Files with .yml.j2 JINJA2 Templates

During Docker deployments, you may need containers with different environment configurations, ports, volumes, etc. Here, I'll discuss how the file type `.yml.j2` leverages the Jinja2 templating engine to simplify this process dynamically. You can dynamically customize your Docker Compose files rather than creating separate files for each different configuration.

A prime example of this is the setup of a **Pi-hole** server.

Here's a typical `docker-compose.yml.j2` file for a Pi-hole deployment:

```yaml
version: "3"
services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "{{ http_port }}:80/tcp"
      - "{{ https_port }}:443/tcp"
    volumes:
       - './etc-pihole/:/etc/pihole/'
       - './etc-dnsmasq.d/:/etc/dnsmasq.d/'
    environment:
      ServerIP: "{{ server_ip }}"
      TZ: "{{ timezone }}"
      WEBPASSWORD: "{{ webpassword }}"
    dns:
      - 127.0.0.1
      - 1.1.1.1
    restart: unless-stopped
```

Here, we have templated the HTTP and HTTPS ports, server IP, timezone, and web password. Its flexibility allows it to cater to users who require different port numbers or differing environmental configurations (e.g., dev, test, prod).

The next part involves the section of the Ansible playbook, which will populate these templates with actual information:

```yaml
- name: Create Docker Compose file from template
  template:
    src: docker-compose.yml.j2
    dest: /path/to/docker-compose.yml
  vars:
    http_port: 80
    https_port: 443
    server_ip: "192.168.1.2" 
    timezone: "America/Toronto"
    webpassword: "mypassword"
```

Ansible replaces the Jinja2 variables with the actual values specified in the vars section of the playbook, resulting in a properly configured docker-compose file ready for a Pi-hole deployment.

## Addressing the error "The following signatures couldn't be verified because the public key is not available: NO_PUBKEY 7EA0A9C3F273FCD8 

Here are the sections of the original script I was using, which produced errors:

```yaml
  - name: Download Docker's official gpg key
    get_url:
      url: https://download.docker.com/linux/ubuntu/gpg
      dest: /etc/apt/keyrings/docker.gpg
      mode: 644

  - name: Add Docker to the sources list
    blockinfile:
      path: /etc/apt/sources.list.d/docker.list
      create: yes
      block: |
        deb [arch={{ dpkg_arch.stdout }} signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu {{ version_codename.stdout }} stable
```

This script consistently produced the error as stated above. After some research, I determined that the issue was with the line(s) specifying "/etc/apt/keyrings/docker.gpg". So, I updated it from "gpg" to "asc". This adjustment seemed to resolve the problem.

Here is the final script for deploying Docker with the required prerequisites for using Ansible to deploy docker-compose files on an Ubuntu host.

`install_docker.yml`

```yaml
- name: Set up Docker
  hosts: newdocker
  become: yes
  tasks:
  - name: Install prerequisites
    apt:
      name: "{{ item }}"
      update_cache: yes
      state: present
    with_items:
      - ca-certificates
      - curl
      - gnupg

  - name: Download Docker's official GPG key
    get_url:
      url: https://download.docker.com/linux/ubuntu/gpg
      dest: /etc/apt/keyrings/docker.asc
      mode: 644

  - name: Retrieve dpkg architecture
    command: dpkg --print-architecture
    register: dpkg_arch

  - name: Retrieve version codename
    shell: . /etc/os-release && echo "$VERSION_CODENAME"
    register: version_codename

  - name: Add Docker to sources list
    blockinfile:
      path: /etc/apt/sources.list.d/docker.list
      create: yes
      block: |
        deb [arch={{ dpkg_arch.stdout }} signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu {{ version_codename.stdout }} stable

  - name: Update apt-get
    apt:
      update_cache: yes

  - name: Install Docker packages
    apt:
      name:
        - docker-ce
        - docker-ce-cli
        - containerd.io
        - docker-buildx-plugin
        - docker-compose-plugin
        - python3-docker
        - python3-pip
      update_cache: yes
      state: latest
  
  - name: Install Docker Compose using pip
    pip:
      name: docker-compose
      state: latest
```

#### Has anyone else had similar issues with GPG versus ASC in apt secure? Leave a comment below!

## Installing Portainer

`portainer-docker-compose.yml.j2`

```yaml
version: "3.3"

services:
  portainer:
    image: portainer/portainer-ce:latest
    restart: always
    network_mode: bridge
    ports:
      - "8000:8000"
      - "9000:9000"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
      - "{{ host_dir }}:/data"
    container_name: portainer

```
`install_portainer.yml`

```yaml
- hosts: "*"
  become: yes
  vars:
    container_name: portainer
    host_dir: "/mnt/docker/portainer/{{ ansible_hostname }}"
    docker_compose_dir: "/home/portainer/"
    ansible_python_interpreter: /bin/python3
  tasks:
    - name: Create directory for nas storage
      file:
        path: "{{ host_dir }}"
        state: directory
    - name: Create directory for docker-compose
      file:
        path: "{{ docker_compose_dir }}/{{ container_name }}"
        state: directory

    - name: create a new docker-compose.yml file
      template:
        dest: "{{ docker_compose_dir }}/{{ container_name }}/docker-compose.yml"
        src: "portainer-docker-compose.yml.j2"

    - name: Run Portainer service using docker-compose
      docker_compose:
        project_src: "{{ docker_compose_dir }}/{{ container_name }}"
        state: present

```

In wrapping up, I walked you through my process of using Ansible and Jinja2 for the installation of Docker and Portainer on an Ubuntu host. I showed you two uses on how Jinja2 can customize Docker Compose files, allowing for flexibility in deploying containers without duplication. 

During the process, I encountered an error related to Docker's official GPG key while using the `install_docker.yml` playbook. The fix was to change "gpg" to "asc" in the respective lines of the playbook!

I hope you found this post insightful. Your feedback is most welcome.