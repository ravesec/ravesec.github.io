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

Debian was installed with the following settings:
```text
IP Address: 192.168.103.200
Netmask: 255.255.255.0
Gateway: 192.168.103.254
Nameserver: 192.168.103.5
Hostname: grafana
Domain: dean.lan
```


