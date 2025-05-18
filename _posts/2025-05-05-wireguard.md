---
title: Exposing Homelab Services to the Internet
date: 2025-05-05 07:30:00 -0500
description: Using Wireguard to configure public access to services.
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

After the configuration files are creafted on both servers, the Wireguard interfaces can be brought up:

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


