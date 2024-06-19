---
title: "Automate Your RKE2 Cluster with Ansible: Helm, Cert-Manager, Traefik, and Rancher Setup Made Easy"
author: isaac
date: 2024-06-19 07:00:00 -0700
categories: [DevOps, Kubernetes, Ansible, Automation, Home Lab]
tags: [Ansible, RKE2, Kubernetes, Helm, Cert-Manager, Traefik, Rancher, Automation, Infrastructure as Code, Cluster Management, Home Lab Projects]
render_with_liquid: false
toc: true
comments: true
image:
  path: /assets/img/myphotos/Automate-Your-RKE2-Cluster-with-Ansible.png
  alt: Automate Your RKE2 Cluster with Ansible
---

## Summary
This Ansible playbook automates the deployment and setup of essential components on an RKE2 cluster, including Helm, Cert-Manager, Traefik, and Rancher. It centralizes variable management to streamline configuration changes and leverages modular roles to ensure tasks are organized and maintainable. By executing this playbook, you can efficiently set up your Kubernetes environment with a consistent and repeatable process.

## The Why
So why are we here? Well, similar to when I would set up a Docker host, I always installed Portainer and Traefik so that subsequent deployments I could easily monitor logs via Portainer and have the new containers with SSL-enabled URLs. My logic was the same for my RKE2 cluster. Using a management tool like Rancher (mostly for monitoring) and as I deploy other pods or containers, I want to be able to leverage Traefik as my reverse proxy to enable valid SSL certificates.

## Getting Started
In my case, I already have the VMs following my previous guide: [Seamlessly Setting Up Server Infrastructure for RKE2 with Semaphore UI (SemUI) and Ansible on Proxmox -- QM Commands!](https://spaceterran.com/posts/Using-SemUI-and-Ansible-To-Build-REK2-Cluster-On-Proxmox/), so I needed a reproducible way of deploying the RKE2 cluster. This is where a great YouTuber (Jim's Garage) came in: `Easy Kubernetes Using Ansible! (RKE2)` [YouTube](https://youtu.be/AnYmetq_Ekc?si=HIajyo5sYiHm7d7Q) [GitHub](https://github.com/JamesTurland/JimsGarage/tree/main/Ansible/Playbooks/RKE2). Frankly, if I cannot deploy something with some sort of CI/CD or DevOps process, I'm not going to do it! There are several other options out there from TechnoTim ([GitHub](https://github.com/techno-tim/k3s-ansible)) or k3sup 🚀 (pronounced 'ketchup') ([GitHub](https://github.com/alexellis/k3sup)), that I could have used, but I just found Jim's to be the absolute minimum, which in my opinion made it very approachable. I don't need an Ansible playbook that can do everything for everyone; I wanted the bare minimum, that was predictable each time and again Jim's playbook does just that! *Just to be clear, the other two work just fine, and I encourage you to check them out. TechnoTim's YouTube channel specifically is a wealth of knowledge if you haven't checked it out before.*

## The Real Getting Started 🙃
First off, if you want to check out the GitHub repo, head on over here: [GitHub Repo](https://github.com/SpaceTerran/ansible-rancher-traefik-ssl). Once you have downloaded all the files to your local environment, the first step you are going to want to do is create your `secrets.yaml` file.

### Create `secrets.yaml`
This `secrets.yaml` is quite simple and only stores the Cloudflare token. This playbook uses the DNS challenge method for validating the Cert-Manager certificates. If DNS challenge validation doesn't mean anything to you or you need more information on the token you need to create or how this works, head over to this: [Cloudflare - cert-manager Documentation](https://cert-manager.io/docs/configuration/acme/dns01/cloudflare/) 
>Note we are using `API Tokens`, not `API Keys`...
 {: .info-tip }

#### Step 1: Initialize the Vault File
To create a new encrypted file, use the `ansible-vault create` command. This will prompt you to choose a password that will be used for encrypting and decrypting the file.
```bash
ansible-vault create secrets.yaml
```
#### Step 3: Add Your Secret
Once you run the `create` command, an editor will open (usually `vi` or `nano`, depending on your system’s configuration). Here, you can add your secrets in YAML format. For instance, to store `CF_TOKEN`:
```yaml
---
CF_TOKEN: "your_cloudflare_token_here"
```
Remember to replace `"your_cloudflare_token_here"` with your actual token value.
#### Step 4: Save and Exit
After adding your secret, save the file and exit the editor. For `vi`, you can do so by pressing `Esc`, typing `:wq`, and then hitting `Enter`.

### Update the `inventory/group_vars/all.yml`
This file is where we store all the variables used in the various roles. Some items to keep in mind below. The home path is just where I chose to store all the files local to one of the RKE2 servers. You may want to put in a different path if you so choose, but that is what the `home_path: /home/adminuser` is all about. This is the home directory of the admin user on the VM host where RKE2 is running. A master node, might I add.

Always check to see what versions are available for a given product. In some cases, or potentially if you are reading this in the future, a later release may break this playbook or you might want to move to a later release to protect yourself from a security vulnerability.

Lastly, one note about the version of Rancher I'm running... First off, it's probably not a good idea to run Alpha in production for a variety of reasons, but for my home lab, I have opted to run the latest version of RKE2 and so at the time of writing this post, the stable and latest didn't support 1.30+, so this is why I opted for the alpha version. You choose what makes most sense for you.

```yml
---
# Home base path
home_path: /home/adminuser

kubectl_config: "{{ home_path }}/.kube/config"

# https://github.com/helm/helm/releases
helm_version: v3.15.1

cert_manager_chart_ref: jetstack/cert-manager
# https://github.com/cert-manager/cert-manager/releases
cert_manager_chart_version: v1.15.0
cert_manager_path: "{{ home_path }}/cert-manager"
cert_manager_email: email@isaacblum.com

traefik_chart_ref: traefik/traefik
# https://github.com/traefik/traefik-helm-chart/releases
traefik_chart_version: 28.3.0
traefik_path: "{{ home_path }}/traefik"

# https://github.com/rancher/rancher/releases
rancher_chart_ref: rancher-alpha/rancher
rancher_chart_version: 2.9.0-alpha5
rancher_path: "{{ home_path }}/rancher"
```

### Update your `inventory/hosts.ini`
Don't forget to update your `hosts.ini` with the servers and their names below. I'd leave the naming, e.g., `[servers]`, alone unless you need and know what else to change in the Ansible playbook. The focus for most, if you are attempting to create a 3 master and 2 worker (agent) deployment, is to just update the IP addresses below, e.g., `192.168.2.126`.

```yml
[servers]
server1 ansible_host=192.168.2.126
server2 ansible_host=192.168.2.125
server3 ansible_host=192.168.2.124

[servers:vars]
ansible_user=adminuser
ansible_become=true

[agents]
agent1 ansible_host=192.168.2.123
agent2 ansible_host=192.168.2.122

[agents:vars]
ansible_user=adminuser
ansible_become=true

```

### Playbook Structure
Not much to say here, besides a visual of what the folder structure looks like. As mentioned, I have broken this out into various roles, with their respective tasks and templates. This will allow for easy updates in the future.

```plaintext
├── inventory/
│   ├── hosts.ini
│   └── group_vars/
│       └── all.yml
├── playbook.yml
├── requirements.yml
├── secrets.yaml
├── roles/
│   ├── common/
│   │   └── tasks/
│   │       └── main.yml
│   ├── helm/
│   │   ├── tasks/
│   │       └── main.yml
│   ├── helm_plugins/
│   │   └── tasks/
│   │       └── main.yml
│   ├── cert_manager/
│   │   ├── tasks/
│   │       └── main.yml
│   │   └── templates/
│   │       ├── cloudflare-api-key-secret.yaml.j2
│   │       └── clusterissuer-letsencrypt-cloudflare.yaml.j2
│   ├── traefik/
│   │   ├── tasks/
│   │       └── main.yml
│   │   └── templates/
│   │       ├── values.yaml.j2
│   │       └── certificate-wildcard-spaceterran-traefik.yaml.j2
│   ├── rancher/
│   │   ├── tasks/
│   │       └── main.yml
│   │   └── templates/
│   │       ├── values.yaml.j2
│   │       └── certificate-wildcard-spaceterran-rancher.yaml.j2
```

### Install Required Ansible Collections

Before running the playbooks, you need to install the required Ansible collections. You can do this by running:

```bash
ansible-galaxy collection install -r requirements.yml
```

## You made it! Now it's time to run the playbook!

### Running the Playbook

To execute the playbook, run:

```bash
ansible-playbook -i inventory/hosts.ini playbook.yml --ask-vault-pass
```


## Conclusion and Feedback
This Ansible playbook simplifies the setup of an RKE2 cluster with Helm, Cert-Manager, Traefik, and Rancher. Whether for a home lab or production environment, it ensures a consistent and efficient deployment process.

I'd love to hear your thoughts and experiences with this playbook. Your comments and feedback are valuable to me!
