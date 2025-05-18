---
title: Exposing Homelab Services to the Internet
date: 2025-05-05 07:30:00 -0500
description: Using Wireguard, Nginx and Cloudflare to configure public access to services.
author: raven
categories: [net399-capstone]
tags: [net399-capstone, extension]
---

# Introduction

This post is part of a larger NET399 capstone course at Eastern Kentucky University and covers the "extension phase" of the project.

The goal of the extension phase is to allow the Grafana instance configured in [the previous posts](http://localhost:4000/categories/net399-capstone/) to be securely accessed from the public internet.

Normally to expose services to the internet all you need to do is port-forward on your router/firewall. However, my ISP is restrictive and with my current network configuration direct port-forwarding will not work. That means that some creativity is required to configure public access to services, which is what I will be covering in this post.

This will be achieved with a combination of different services:
1. Cloudflare for DNS
2. OVHCloud for a cloud hosted VPS
3. Proxmox for VM hosting
3. Wireguard to create a tunnel from a cloud server to my internal network
4. Nginx Proxy Manager to secure access to services with an access list

Secure public access to my Grafana instance can be broken down into these steps:
1. Obtain access to a cloud server and create a dedicated lxc container on my private network
2. Configure Wireguard on both servers
3. Configure a firewall on the cloud server
4. Verify connectivity between the cloud server and the Grafana server
5. Install and configure Nginx proxy manager on the cloud server
6. Configure a DNS entry for Grafana using Cloudflare
7. Create a read-only account on Grafana for public access
8. Generate an SSL certificate using Let's Encrypt for secure access to the Grafana instance
9. Create and apply the access list on Nginx Proxy Manager

Now, let's get started.

## Cloud Server and Virtual Machine

A while ago I purchased a VPS using [OVHCloud](https://us.ovhcloud.com/vps/) to use for this purpose. I also created a dedicated Wireguard lxc container on Proxmox. Here are screenshots of both servers:

![Desktop View](/assets/img/proxmox-lxc-container.png)
_The private LXC on Proxmox._

![Desktop View](/assets/img/ovhcloud-vps.png)
_The VPS hosted on OVHCloud._

## Configuring Wireguard

I'll configure Wireguard to allow the VPS to access the Grafana instance.

### Generating Keys

Wireguard uses public key cryptography to secure communication between hosts. The first step to configuring the tunnel is to generate private and public keys on the hosts.

The goal is to have configure a tunnel between the VPS and container with the addresses 10.40.0.1/24, and 10.40.0.2/24 and be able to ping 192.168.103.200 (Grafana) from the VPS.

```bash
# Generate private key
wg genkey | tee privatekey

# Derive public key from private key
wg pubkey < privatekey | tee publickey
```

```bash
# On both servers
vim /etc/wireguard/wg0.conf
```

### Interface Configuration

wg0 configuration on the VPS
```text
[Interface]
Address = 10.40.0.1/24
ListenPort = 52920
PrivateKey = <vps private key>

[Peer]
PublicKey = <public key of container>
AllowedIPs = 192.168.103.200/32,10.40.0.2/32
```

wg0 configuration on the container
```text
[Interface]
Address = 10.40.0.2/24
PrivateKey = <container's private key>

PostUp = sysctl -w net.ipv4.ip_forward=1 && iptables -t nat -A POSTROUTING -o eth0 -s 10.40.0.0/24 -j MASQUERADE && iptables -A FORWARD -i wg0 -o eth0 -j ACCEPT && iptables -A FORWARD -i eth0 -o wg0 -m state --state RELATED,ESTABLISHED -j ACCEPT

PostDown = sysctl -w net.ipv4.ip_forward=0 iptables -t nat -D POSTROUTING -o eth0 -s 10.40.0.0/24 -j MASQUERADE && iptables -D FORWARD -i wg0 -o eth0 -j ACCEPT && iptables -D FORWARD -i eth0 -o wg0 -m state --state RELATED,ESTABLISHED -j ACCEPT

[Peer]
PublicKey = <public key of VPS>
Endpoint = 15.x.x.x:52920
AllowedIPs = 10.40.0.1/32
PersistentKeepAlive = 25
```

The wg0 interface configuration on the container enables SNAT translation when the interface comes up, so that on my private network all traffic coming from the outside appears as normal traffic from the container. This is important so that exposed services will know how to send traffic back to the source.

After the configuration files are created on both servers, the Wireguard interfaces can be brought up:

### Starting Interfaces and Verifying Connectivity

```bash
# Bring up wg0
wg-quick up wg0

# Verify the status of the tunnel
wg show

# Verify the connection between both tunnel endpoints
ping 10.40.0.1 -c 4
ping 10.40.0.2 -c 4
ping 192.168.103.200 -c 4
```

![Desktop View](/assets/img/wireguard-connectivity-check.png)
_Output of the wg show command and ping tests showing successful connectivity._

## Configuring Nginx Proxy Manager

> nginx ("engine x") is an HTTP web server, reverse proxy, content cache, load balancer, TCP/UDP proxy server, and mail proxy server.

I will be using Nginx to manage access to my services. By configuring the proxy manager, I will be able to route traffic to specific IP addresses by DNS names. The goal of this step is to route traffic to https://grafana.ravenn.net:443 to https://192.168.103.200:443.

### Installation

I installed Nginx Proxy Manager using Docker, which itself can be installed in a few steps;

```bash
# Uninstall any conflicting packages
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove $pkg; done

# Set up Docker's apt repository
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

# Install the latest Docker version
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Verify that the installation is working
sudo docker run hello-world
```

Now that Docker is installed, the next step is to create a docker compose file for Nginx Proxy Manager.

```bash
# Go home
cd ~

# Create a directory for the files and cd into it
mkdir nginx-proxy-manager

# Create the docker compose file
vim docker-compose.yml
```

Contents of `docker-compose.yml`:
```yml
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    ports:
      # These ports are in format <host-port>:<container-port>
      - '80:80' # Public HTTP Port
      - '443:443' # Public HTTPS Port
      - '8081:81' # Admin Web Port
      # Add any other Stream port you want to expose
      # - '21:21' # FTP

    environment:
      # Uncomment this if you want to change the location of
      # the SQLite DB file within the container
      # DB_SQLITE_FILE: "/data/database.sqlite"

      # Uncomment this if IPv6 is not enabled on your host
      DISABLE_IPV6: 'true'

    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
```
```bash
# Bring up the proxy manager
sudo docker compose up -d
```
Now that the container is up, the proxy manager can be accessed on the admin web port. Upon logging in with the default credentials `admin@example.com:changeme`, you will need to change the credentials.

![Desktop View](/assets/img/nginx-proxy-manager-login-page.png)
_The Ngnix Proxy Manager Login page._

### Configuration

Now we can access the admin panel of the proxy manager, but access is insecure. It can be accessed publicly over http with no authorization. We can fix this by proxying the service itself, removing public access to the admin port and creating an access list.

#### Creating a Proxy Host

After logging in to the proxy manager, you will see the main landing page:

![Desktop View](/assets/img/nginx-proxy-manager-landing-page.png)
_Nginx Proxy Manager landing page._

Here, we can configure Proxy Hosts, Redirection Hosts, Streams, and 404 Hosts. In this post I will only be worrying about proxy hosts.

1. Navigate to the proxy hosts page and click Add Proxy Host. You will see a new proxy host creation window.

![Desktop View](/assets/img/nginx-proxy-manager-new-proxy-host-window.png)
_Window for creating a new proxy host._

For the domain name, I chose proxy.ravenn.net. The admin page is hosted on port 81 inside the docker container, so for the forward hostname / IP I set 127.0.0.1:81. For now, leave the Access List setting as 'Publicly Accessible'.

![Desktop View](/assets/img/nginx-proxy.ravenn.net-proxy-host-settings.png)
_Settings for proxy host proxy.ravenn.net_

Hit save.

2. Create the DNS entry in Cloudflare

Next I navigated to Cloudflare's dashboard for managing the ravenn.net domain.

I created a new DNS address record pointing to proxy.ravenn.net. By using the proxy option, traffic will be first proxied through Cloudflare automatically providing an SSL certificate for proxy.ravenn.net.

Unfortunately, proxying traffic through cloudflare can cause http basic authentication to fail, so we will use another method for SSL access later on.

![Desktop View](/assets/img/cloudflare-dns-record-proxy.ravenn.net.png)
_A Record for proxy.ravenn.net_

After a minute or so, we can validate that the DNS record exists:

```bash
dig proxy.ravenn.net
```

![Desktop View](/assets/img/output-of-dig-command.png)
_dig command showing that proxy.ravenn.net points to Cloudflare's servers_

4. Verifying the Setup

Now we should be able to navigate to https://proxy.ravenn.net and see that the proxy is working correctly, but the connection is still not secure.

![Desktop View](/assets/img/proxy.ravenn.net-insecure-connection.png)
_http://proxy.ravenn.net successfully redirects to the login page._

5. Removing direct access to the admin port

Currently the admin page can still be directly accessed over port 8081. We can fix that by taking down the docker container, commenting out the line that exposes the admin page, and bringing it back up.

```bash
# Navigate to the folder containing the proxy manager files
cd ~/nginx-proxy-manager

# Take down the container
docker compose down

# Edit the container's compose file
vim docker-compose.yml
```

Comment out the line that exposes port 81 on the docker container to port 8081 on the VPS.

```yml
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    ports:
      # These ports are in format <host-port>:<container-port>
      - '80:80' # Public HTTP Port
      - '443:443' # Public HTTPS Port
      # - '8081:81' # Admin Web Port
      # Add any other Stream port you want to expose
      # - '21:21' # FTP

    environment:
      # Uncomment this if you want to change the location of
      # the SQLite DB file within the container
      # DB_SQLITE_FILE: "/data/database.sqlite"

      # Uncomment this if IPv6 is not enabled on your host
      DISABLE_IPV6: 'true'

    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
```

```bash
# Bring the service back uo
docker compose up -d
```

Now only ports 80 and 443 are exposed, and the admin panel is only served over localhost on Docker.

When we try to navigate to http://proxy.ravenn.net:8081, the page will never load. The unsecure connection message is just due to caching by Cloudflare.

![Desktop View](/assets/img/nginx-proxy-manager-direct-access-blocked.png)
_Admin page is no longer directly accessible over port 8081._

6. Configuring Authorization for the Proxy Manager

The final step in securing access to the admin page is to create an access list so that navigating to https://proxy.ravenn.net requires a password.

We can do this by logging in to the proxy manager and navigating to `Access Lists`. Once we are there, a new access list can be created by pressing the 'Add Access List' button.

Enter a name for the access list. I chose 'proxy-admin-panel'.

Navigate to the Authorization tab and create a username and password for the access list. The user and pass should be the same as the credentials you configured for the admin panel, allowing you to log in with HTTP basic authentication.

Navigate to Access and configure access for specific subnets. I just specified 0.0.0.0/0.

![Desktop View](/assets/img/nginx-proxy-manager-admin-access-list.png)
_Newly Created Access List_

7. Go back and apply the new access list to the proxy host.

Once the access list is applied, you are immediately greeted with a basic http authentication prompt.

![Desktop View](/assets/img/nginx-proxy-manager-auth-page.png)
_Sign in request._

8. Requesting a Certificate from Let's Encrypt

Finally I can request my own certificate by using Let's Encrypt. Navigate to the Proxy Host and go to the SSL tab. Under SSL certificate, choose "Request a new SSL certificate". You will be asked to provide an email address and agree to the Let's Encrypt terms of service. Once you do that, hit save, and after a few moments Let's Encrypt will grant a new SSL certificate.

Now the SSL certificate shows up under the SSL Certificates tab in the proxy manager, and the connection is secured.

![Desktop View](/assets/img/proxy.ravenn.net-secure-connection.png)
_Access to proxy.ravenn.net over HTTPS_

![Desktop View](/assets/img/proxy.ravenn.net-certificate.png)
_Certificate provided to proxy.ravenn.net from Let's Encrypt_


## Making Grafana Publicly Accessible

Now that everything is configured, making the Grafana instance publicly accessible via https://grafana.ravenn.net is trivial:

1. Create a DNS A record for grafana.ravenn.net
2. Create a proxy host pointing to the Grafana instance, so that https://grafana.ravenn.net -> https://192.168.103.200
3. Generate a certificate for grafana.ravenn.net using Let's Encrypt.
4. Create an access list for https://grafana.ravenn.net

Now, Grafana is accessible to the open internet.

![Desktop View](/assets/img/nginx-proxy-manager-final-proxy-hosts.png)
_Proxy hosts_

![Desktop View](/assets/img/nginx-proxy-manager-final-ssl;-certificates.png)
_SSL certificates_

![Desktop View](/assets/img/nginx-proxy-manager-final-access-lists.png)
_Access Lists_

![Desktop View](/assets/img/grafana.ravenn.net.png)
_https://grafana.ravenn.net_

Finally, I need to create a read-only user for Grafana. Initially you may think it is strange to allow anyone to access Grafana. Here is a quote from their website:

> Data everyone can see
Grafana was built on the principle that data should be accessible to everyone in your organization, not just the single Ops person. By democratizing data, Grafana helps to facilitate a culture where data can easily be used and accessed by the people that need it, helping to break down data silos and empower teams.

Of course, the Grafana access here is for capstone purposes only. The credentials will be valid for a limited time (about a month).

### Creating the Read Only user

On the Grafana instance, navigate to `Home > Administration > Users and Access > Users` and create a new user.

I created a new user with the same credentials as the access list on the proxy manager; `capstone:capstone`.

![Desktop View](/assets/img/grafana-view-only-user.png)
_Creating the capstone user._

By default, the user will be provided with view-only permissions only.

The Grafana instance is now accessible to the public, and a view-only user is able to access it with the credentials `capstone:capstone`.

![Desktop View](/assets/img/signing-in-to-grafana.png)
_Signing in as the capstone user_

![Desktop View](/assets/img/grafana-view-only-access.png)
_The main Grafana page with read only permissions._

![Desktop View](/assets/img/grafana-view-only-openwrt-dashboard.png)
_A dashboard publicly accessible to the view-only user._

# Summary

In this post, I configured my Grafana instance to be publicly accessible to the internet. I went over creating DNS records in Cloudflare, installing Nginx Proxy Manager using Docker and configuring it, and creating a Wireguard tunnel to expose internal routes to public facing servers.
