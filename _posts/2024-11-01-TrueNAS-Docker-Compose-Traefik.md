---
title: "Hands-On with TrueNAS SCALE 24.10 Electric Eel: Configure Docker Compose and Traefik - Working SSL Certificates"
author: isaac
date: 2024-09-30 07:00:00 -0700
categories: [TrueNAS, Docker, Docker Compose, Networking, HomeLab, SSL Certificates, Tutorials and Guides]
tags: [TrueNAS SCALE, TrueNAS SCALE 24.10, Electric Eel, Docker Compose, Traefik, SSL Certificates, Home Server, Portainer, Cloudflare]
render_with_liquid: false
toc: true
comments: true
image:
  path: /assets/img/myphotos/truenas_docker_compose_traefik/TrueNAS_SCALE_24.10_Electric_Eel_Docker_Traefik_SSL.png
  alt: 
---

## TrueNAS SCALE 24.10 (Electric Eel)

If you're like me, you want to start using Docker Compose right away on the new release of TrueNAS SCALE 24.10 (Electric Eel). The main reason is that I already have a ton of Docker Compose files that allow me to host my homelab on almost any server, as long as it supports Docker. 

## Traefik

I am also a fairly big fan of trusted SSL certificates for my various self-hosted applications. I use Traefik, thanks to [Christian Lempa](https://www.youtube.com/@christianlempa) way back when, who brought it to my attention initially. Once I deployed it once, I never wanted to go back!!! While this is just a few days from the release of TrueNAS Electric Eel, when I went to the Apps catalog in TrueNAS it wasn't listed. 
 
Again, these are early days and it's quite possible that by the time you read this, it will be a published application by iXsystems, but I didn't want to wait. I had workloads I wanted to move over!!!!!

![impatient](/assets/img/myphotos/truenas_docker_compose_traefik/impatient.png){: width="200" .center } 

Additionally, it looks like Docker Compose support, again in the early days, really isn't available within the GUI. Or maybe I'm blind... Either way, I didn't want to wait. This tutorial is, well, breaking all the rules... making changes directly to the TrueNAS OS, and will most likely make your instance not supportable. And frankly, you probably shouldn't do this... But if you're like... **SEND IT!!!** then keep on reading.

![send it](/assets/img/myphotos/truenas_docker_compose_traefik/sendit.png){: width="200" .center }

## Enabling SSH Access

Start off by enabling SSH access to your TrueNAS. You don't need this step strictly speaking, but it's a bit nicer to use your terminal of choice versus the web interface.

### Option 1 (Recommended):

1. Log in as normal to your TrueNAS instance.
2. Navigate to *System*, then *Services*, and finally tick the box to enable the SSH service if not already running.  
   ![SSH Service Toggle](/assets/img/myphotos/truenas_docker_compose_traefik/SSH_Service_Toggle.png){: width="400" .normal }  
   ![SSH Enabled Notification](/assets/img/myphotos/truenas_docker_compose_traefik/SSH_Enabled_Notification.png){: width="400" .normal }

3. Lastly, ensure your admin account has SSH login access. Navigate to *Credentials -> Users -> Admin*, then select *Edit*.  
   ![Admin User SSH Permissions](/assets/img/myphotos/truenas_docker_compose_traefik/Admin_User_SSH_Permissions.png){: width="400"}

> **Ensure you turn this off once you are done!** It's not the best idea to leave SSH running and allowing remote access to your TrueNAS unless you have to.
{: .prompt-danger }

### Option 2:

Navigate to *System* then *Shell*.  
![TrueNAS Shell Access](/assets/img/myphotos/truenas_docker_compose_traefik/TrueNAS_Shell_Access.png){: width="400" .normal }

## GUI Changes for Ports

Now we need to make a couple of changes in the GUI. It's very possible these could be command line items, but you know... sometimes it's just quicker, so here we go.

### Changing Default Ports

1. Navigate to *System -> General Settings* -> Under *GUI*, select *Edit*.  
   ![TrueNAS Port Settings](/assets/img/myphotos/truenas_docker_compose_traefik/TrueNAS_Port_Settings.png){: width="400" .normal }

2. Set port 80 to 8080 and 443 to 4433. Remember, after you hit save, you will need to change the port in your URL.  
   ![Port Change Confirmation](/assets/img/myphotos/truenas_docker_compose_traefik/Port_Change_Confirmation.png){: width="400" .normal }

3. Confirm and continue when prompted to reboot the web services. Note that restarting the web services may take up to a minute, so be patient.  
   ![Web Service Reboot Confirmation](/assets/img/myphotos/truenas_docker_compose_traefik/Web_Service_Reboot_Confirmation.png){: width="400" .normal }

4. Update your URL; in my case, it changed from `https://192.168.1.150` to `https://192.168.1.150:4433`. You can ignore SSL warnings at this point.

## Installing Docker

### App Configuration

1. Select *App* from the sidebar, then the down arrow on *Configuration*.  
   > **Note:** I'm assuming you have never installed any TrueCharts and you have a fresh install of TrueNAS. If this isn't the case, your screens may not be in the same state as mine.
   {: .prompt-info }

2. Choose a pool.  
   ![App Pool Selection](/assets/img/myphotos/truenas_docker_compose_traefik/App_Pool_Selection.png){: width="400" .normal }

3. In my case, I chose my spinning disks. I'm only planning on running Traefik and a couple of smaller containers like Pi-hole or Plex, so placing this on my faster storage is just not needed. Select Choose.
  ![](/assets/img/myphotos/truenas_docker_compose_traefik/Pool_Selection_Spinner_Disks.png)
  ![](/assets/img/myphotos/truenas_docker_compose_traefik/Pool_Selection_Choose.png)

3. Once you see the green check box, you are ready to head back to the Shell.  
   ![Docker Installation Green Check](/assets/img/myphotos/truenas_docker_compose_traefik/Docker_Installation_Green_Check.png){: width="400" .normal }

## Locating the Docker Installation

Login to SSH. Now that you have SSH'ed into your shell, you will want to elevate to SUDO for the rest of the commands. This is accomplished by the following command. When asked for a password, ensure you put in your admin password. *Note that typing the password is not needed if you selected "allow all sudo commands with no password."*

```bash
sudo su
```

Your screen should look like this:  
![TrueNAS Shell Sudo Su](/assets/img/myphotos/truenas_docker_compose_traefik/TrueNAS_Shell_Sudo_Su.png){: width="400" .normal }

Run the following commands to confirm the docker directory and navigate:

```bash
ls /mnt/.ix-apps/
cd /mnt/.ix-apps/docker
```

![Docker Directory Check](/assets/img/myphotos/truenas_docker_compose_traefik/Docker_Directory_Check.png)

### Folder Structure

Now I chose to place a new folder contained in the Docker installation directory. This may not be wise long-term as upgrades could cause issues, but for now, this is where I placed it. I created a folder for MyApps and the folder structure for Traefik and Portainer for the demonstration.

Create folders for your applications:

```bash
mkdir MyApps
mkdir MyApps/traefik
mkdir MyApps/traefik/config
mkdir MyApps/traefik/logs
mkdir MyApps/traefik/sslcerts
mkdir MyApps/portainer
mkdir MyApps/portainer/data
```

![MyApps Folder Creation](/assets/img/myphotos/truenas_docker_compose_traefik/MyApps_Folder_Creation.png)

>Note: I'm just using Portainer; as a test application, to show SSL working in a later step. Feel free to omit this.
{: .prompt-warning }

### Configuring Traefik

Now the following is quite manual. I highly recommend using something like Ansible or using SSH to push the files from your local to these directories, but for these demonstrations, I will just show you each file that I create and provide the contents of each configuration file. Again, this is just a demonstration, so you may have other needs and can update the files as you see fit.

Navigate to:

```bash
cd /mnt/.ix-apps/docker/MyApps/Traefik
```

Create and edit the `docker-compose.yaml` file:

```bash
nano docker-compose.yaml
```

#### Contents of `docker-compose.yaml`

```yaml
services:
  traefik:
    image: traefik:latest
    container_name: traefik
    security_opt:
      - no-new-privileges:true    
    ports:
      - "80:80" # Traefik 
      - "443:443" # Traefik
      - "8181:8080" # Traefik Dashboard # I recommend removing it once you prove the system is working.
    networks:
      - proxy_network      
    volumes:
      - type: bind
        source: /mnt/.ix-apps/docker/MyApps/traefik/config # These are the paths I chose, but I suspect they can be placed elsewhere.
        target: /etc/traefik
      - type: bind
        source: /mnt/.ix-apps/docker/MyApps/traefik/sslcerts # These are the paths I chose, but I suspect they can be placed elsewhere.
        target: /etc/traefik/sslcerts
      - type: bind
        source: /mnt/.ix-apps/docker/MyApps/traefik/logs # These are the paths I chose, but I suspect they can be placed elsewhere.
        target: /var/log/traefik/
      - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
      - CLOUDFLARE_API_KEY=Your_Key_Here
      - CLOUDFLARE_EMAIL=Your_Email_Here
      - CLOUDFLARE_PROPAGATION_TIMEOUT=90 # I never needed this before, but for some reason, I needed it on TrueNAS.
    restart: unless-stopped

networks:
  proxy_network:
    external: true

```

Navigate to the config directory and create two files:

```bash
cd config
nano traefik.yaml
```

#### Contents of `traefik.yaml`

```yaml
global:
  checkNewVersion: true
  sendAnonymousUsage: false

log:
 level: INFO
 format: common
 filePath: /var/log/traefik/traefik.log

accessLog:
  format: common
  filePath: /var/log/traefik/access.log

# Comment this section out after you have completed testing.
api:
  dashboard: true
  insecure: true
# Comment this section out after you have completed testing.

entryPoints:
  web:
    address: :80
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
  websecure:
    address: :443

certificatesResolvers: # Note I'm using Cloudflare, but if you want to use a different type of challenge, then you will need to update this section.
  production:
    acme:
      email: Your_Email_here@gmail.com
      storage: /etc/traefik/sslcerts/acme.json
      caServer: "https://acme-v02.api.letsencrypt.org/directory"
      dnsChallenge:
        provider: cloudflare
        resolvers:
          - "1.1.1.1:53"

serversTransport:
  insecureSkipVerify: true

providers:
  docker:
    exposedByDefault: false
    network: proxy_network
  file:
    directory: /etc/traefik
    watch: true
```

Create the `dynamic.yaml` file:

```bash
nano dynamic.yaml
```

#### Contents of `dynamic.yaml`

```yaml
http:
  routers:               

  services:

  middlewares:
    default-headers:
      headers:
        browserXssFilter: true
        contentTypeNosniff: true
        forceSTSHeader: true
        stsIncludeSubdomains: true
        stsPreload: true
        stsSeconds: 15552000
```

Move back one directory:

```bash
cd ..
```

![Traefik Directory Back](/assets/img/myphotos/truenas_docker_compose_traefik/Traefik_Directory_Back.png)

## Creating External Network

Now you will need to create the external network since, in the Docker Compose file, it's listed as external. Both Traefik and the test Portainer image you will use for testing will share this network. Remember, the name here must match the docker-compose.yaml file you created or edited in the previous step.

```bash
docker network create proxy_network
```
![proxy_network](/assets/img/myphotos/truenas_docker_compose_traefik/proxy_network.png)

Spin up Traefik:

```bash
docker compose up -d
```

![started](/assets/img/myphotos/truenas_docker_compose_traefik/started.png)

Now open a web browser and navigate to the Traefik dashboard at `http://192.168.1.150:8181`. *Note: You need to update the IP based on the IP of your TrueNAS.* Port 8181 is defined in the Docker Compose file and should be turned off if not in use for security reasons. This is something you can put behind a password, but that is out of scope for this post.

![dashboard](/assets/img/myphotos/truenas_docker_compose_traefik/dashboard.png)

If your screen looks like the above, you have a Docker Compose Traefik instance running on TrueNAS SCALE 24.10 (Electric Eel), ready to pull in static or Docker-based labels for SSL certificates. 

Next, I will share my Docker Compose for Portainer, just so you can see the labels I use. I'm not going to go into details about it. Note that I'm also not covering DNS here. Once you set your compose file up for a domain(traefik.http.routers.frontend.rule=Host), you need to point your DNS to the IP of the TrueNAS server, so that Traefik can not only serve the site but also allow for the validated certificate to show. If you need DNS help, I'm happy to assist—just leave me a line in the comments.

## Portainer

Here is the compose file I used.

```yaml
services:
  portainer:
    image: portainer/portainer-ce:sts
    container_name: portainer
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /mnt/.ix-apps/docker/MyApps/portainer/data:/data
    networks:
      - proxy_network 
    restart: unless-stopped                       
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.frontend.rule=Host(`demo1.spaceterran.com`)"
      - "traefik.http.routers.frontend.entrypoints=websecure"
      - "traefik.http.services.frontend.loadbalancer.server.port=9000"
      - "traefik.http.routers.frontend.service=frontend"
      - "traefik.http.routers.frontend.tls.certresolver=production"

networks:
  proxy_network:
    external: true

volumes:
  data:
    driver: local
```

Here is a screenshot of the site with a working SSL certificate!

![working_site](/assets/img/myphotos/truenas_docker_compose_traefik/working_site.png)

## Closing
Thanks for diving into the TrueNAS SCALE 24.10 (Electric Eel) setup with me! 🐍⚡ With Docker Compose and Traefik! 

Got questions or cool tweaks to share? Jump into the comments and let’s chat! Sharing insights and tales helps me ensure these articles are making a difference. 

If you enjoyed this guide, why not pass it along to a fellow tech enthusiast? 

See you in the comments! 🚀💬