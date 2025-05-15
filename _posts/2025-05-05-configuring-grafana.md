---
title: Network Visualization using Grafana
date: 2025-05-05 07:30:00 -0500
description: "Dashboard anything. Observe everything."
author: raven
categories: [net399-capstone]
tags: net399-capstone
---

# What is Grafana?

Grafana is a data visualization and analytics platform that allows you to query, visualize, and alert on data with the use of customizable dashboards.

Grafana can be used to visualize any kind of data no matter where it is stored. Below are some examples of different dashboards / data visualizations that are possible with Grafana that can be found on the [official website](https://grafana.com/grafana/).

## Dashboard Examples

![Desktop View](/assets/img/grafana-kubernetes-dashboard-example.png)
_A custom dashboard showing Kubernetes cluster capacity data_

![Desktop View](/assets/img/grafana-home-energy-dashboard-example.png)
_A custom dashboard showing home energy consumption and cost_

![Desktop View](/assets/img/grafana-website-performance-dashboard-example.png)
_A custom dashboard showing metrics from a website_

![Desktop View](/assets/img/grafana-team-sprints-dashboard-example.png)
_A custom dashboard showing team sprints_

![Desktop View](/assets/img/grafana-revenue-dashboard-example.png)
_A custom dashboard showing revenue predictions for business intelligence_

## Network Visualization

For this project, I will be using Grafana to visualize information from devices and servers in my homelab, including a Palo Alto firewall, a Cisco Switch, and OpenWRT access point, and two Proxmox servers. I will be using [community made dashboards](https://grafana.com/grafana/dashboards/) to achieve this.

# Getting Started
## Relevant Links
https://grafana.com/docs/grafana/latest/setup-grafana/installation/
https://grafana.com/docs/grafana/latest/setup-grafana/installation/debian/

Grafana offers a forever free plan that lets you host Grafana in the cloud, however I will be self-hosting my instance. Grafana supports Debian, Ubuntu, RHEL/Fedora, SUSE/openSUSE, macOS, and Windows. For my installation I will be using Debian.

## Creating the Virtual Machine

I will be using Proxmox to host Grafana. 

### General

![Desktop View](/assets/img/creating-grafana-vm-general.png)

I am creating my VM on my second proxmox server since it already has the Debian 12 install CD downloaded and ready to use. I named the VM Grafana, assigned it a VM ID of 200, and made it start on boot.

### OS

![Desktop View](/assets/img/creating-grafana-vm-os.png)

For the OS settings, I am using a Debian 12.9 install CD with Linux 6.x-2.6 kernel for the guest OS.

### System

![Dektop View](/assets/img/creating-grafana-vm-system.png)

For the system settings, I enabled the QEMU agent and left all other options default.

### Disks

![Desktop View](/assets/img/creating-grafana-vm-disks.png)

For the disk settings, I gave the VM 32GB of storage to start with, as it can be increased later if needed. Additionally I enabled discard and write-back cache mode for increased VM performance.

### CPU

![Desktop View](/assets/img/creating-grafana-vm-cpu-1.png)
_Allocation Section_

![Desktop View](/assets/img/creating-grafana-vm-cpu-2.png)
_Extra CPU Flags Section_

#### Allocation
Grafana requires a minimum of 1 CPU core. For my install I will give it 4 cores spanning two sockets to match the physical configuration of the host. I also enabled NUMA, and changed the CPU type to "Host".

#### CPU Flags
Due to the old hardware that my Proxmox server is running on, it is necessary for me to disable the AES instruction set to get VMs to work. I also disable 1GB size pages (pdpe1gb) for increased performance. Everything else can remain default.

### Memory

![Desktop View](/assets/img/creating-grafana-vm-memory.png)

Grafana requires a minimum of 512MB memory. I will enable balooning to save resources, then set the minimum memory to 512MiB and the max memory to 6144MiB.

### Network

![Desktop View](/assets/img/creating-grafana-vm-network.png)

For the network settings, I will attach this VM to my Trusted Servers network (VLAN 40) and disable the firewall. All other settings remain default.

## Installing Debian

Debian was installed headless and with the following settings:
```text
IP Address: 192.168.103.200
Netmask: 255.255.255.0
Gateway: 192.168.103.254
Nameserver: 192.168.103.5
Hostname: grafana
Domain: dean.lan
```
## Installing Grafana

![Desktop View](/assets/img/grafana-vm-shell.png)
_A shell on the new Grafana VM._

Now that we have access to a shell, we can start the installation process by following the offical [installation instructions](https://grafana.com/docs/grafana/latest/setup-grafana/installation/debian/) for Debian.

### Step 1: Installing packages

```bash
sudo apt-get install -y apt-transport-https software-properties-common wget
```

### Step 2: Import the GPG key

```bash
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
```

### Step 3: Add the repository, Update the list of available Packages and Install the latest OSS release of Grafana

The installation guide gives instructions for adding the stable and the beta repository for Grafana as well as installing the enterprise version of Grafana. For this install I will choose the stable OSS release.

```bash
# Add the repository for stable releases
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list

# Update the list of available packages
sudo apt-get update

# Install the latest OSS release of Grafana
sudo apt-get install grafana
```

## Setting up the Grafana Server
### Relevant Links
https://grafana.com/docs/grafana/latest/setup-grafana/start-restart-grafana/
https://grafana.com/docs/grafana/latest/setup-grafana/configure-grafana/


Next we will start the server and configure it to start at boot with systemd.

### Step 1: Start the server and confirm that it is running

```bash
# Start the service
sudo systemctl daemon-reload
sudo systemctl start grafana-server

# Verify that the service is running
sudo systemctl status grafana-server
```

![Desktop View](/assets/img/grafana-service-is-running.png)
_Output showing the status of grafana-server._

### Step 2: Serve Grafana on a port less than 1024 (Optional)

By default, the Grafana server listens on port 3000. Later I will configure the server to listen on port 443, so I will need to add a systemd unit override to grant the CAP_NET_BIND_SERVICE capability.

```bash
# Edit the service
sudo systemctl edit grafana-server.service
```

```ini
[Service]
# Give the CAP_NET_BIND_SERVICE capability
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_BIND_SERVICE

# A private user cannot have process capabilities on the host's user
# namespace and thus CAP_NET_BIND_SERVICE has no effect.
PrivateUsers=false
```

```bash
# Restart the Grafana server
sudo systemctl restart grafana-server
```

### Step 3: Configure Grafana

Next, I will configure some settings for the Grafana instance. My Grafana configuration file is located at `/etc/grafana/grafana.ini`. All values in this file have default values, so I only need to uncomment things I want to change.

The changes I made to the server are below:

```ini
# /etc/grafana/grafana.ini

[server]
# Protocol changed to https from http
protocol = https

# Minimum TLS value is set to TLS1.2
min_tls_version = "TLS1.2"https://grafana.com/docs/grafana/latest/setup-grafana/configure-grafana/

# The port is changed to 443 from 3000
http_port = 443

# The FQDN is set
domain = grafana.dean.lan
```

Make sure to restart the server to apply the settings and verify that the server is running.

```bash
sudo systemctl restart grafana-server.service
sudo systemctl status grafana-server.service

# Ensure Grafana is listening on the correct port
sudo ss -luntp
```

We should now be able to access the server over https after accepting the security warning:

![Desktop View](/assets/img/grafana-login-page.png)
_The Grafana login page._

After logging in with the default credentials `admin:admin`, you will be warned about using default credentials and asked to change them.

![Desktop View](/assets/img/grafana-default-credential-warning.png)
_Warning about using default credentials._

After changing the credentials (or skipping), you are redirected to the welcome page.

![Desktop View](/assets/img/grafana-welcome-page.png)
_The Grafana welcome page._

# Customizing Grafana
