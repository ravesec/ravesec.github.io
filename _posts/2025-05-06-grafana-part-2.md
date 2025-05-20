---
title: Network Visualization using Grafana - Part 2
date: 2025-05-06 07:30:00 -0500
description: "Configuring alerting using Grafana"
author: raven
categories: [net399-capstone]
tags: [net399-capstone, core2]
---

# Introduction

This post is part of a larger NET399 capstone course at Eastern Kentucky University and covers "core phase 2" of the project.

In the [previous post](https://ravesec.github.io/posts/grafana), I installed and configured Grafana and Prometheus to visualize metric data from two Proxmox servers, an OpenWRT access point, and a Pi-hole DNS server.

The goal of core phase two is to configure Grafana to send alerts based on metric data from critical infrastructure in my homelab. Alerts can be sent to many services by using webhooks, and in this post I will be using Discord webhooks to receive alert messages from my Pi-hole DNS server metrics.

## Relevant Links
<https://grafana.com/docs/grafana/latest/alerting/>
<https://grafana.com/docs/grafana/latest/alerting/configure-notifications/manage-contact-points/integrations/configure-discord/>

# Getting Started with Alerting using Grafana

Grafana alerting works by periodically querying data sources and evaluates a condition defined in an alert rule. Then, If the condition is breached, an alert instance fires. Firing alert instances are routed to notification policies based on matching labels. Notifications are then sent out to contact points specified in the notification policy.

## Configuring a Contact Point

To receive alerts, the first thing we must do is create a contact point. The first step here is to navigate to a Discord server where you have admin rights and generate a webhook link using the integrations tab.

1. Open the server settings, and navigate to `Apps > Integrations > Webhooks`.
2. Press the 'New Webhook' button, and Discord will create a webhook bot for you. You can click the bot and customize the name, profile image, and channel. I named my bot 'grafana-alerts' and set it to send to a channel I created called 'grafana-alerts'.

![Desktop View](/assets/img/discord-integrations-page.png)
_The newly created webhook bot_

3. Copy the webhook link that was generated for the new bot.
4. Open your Grafana instance, and navigate to `Home > Alerting > Contact Points`.
5. Press the '+ Create contact point button'. Now you can edit the contact point.
6. Choose a name for the contact point. I named mine 'Discord'. For the integration, hit the dropdown and select 'Discord', then paste the webhook URL.
7. Hit 'Test' to test the webhook connection. You should see a test message appear in the Discord channel you specified. Finally, save the contact point.

![Desktop View](/assets/img/grafana-alerting-contact-point.png)
_The new contact point for Discord_

![Desktop View](/assets/img/grafana-alerting-test-message.png)
_A successful test message, confirming that the webhook is working._

## Configuring the Alert rule

Now that we have a working contact point, we need to configure an alert rule to use it. On the Grafana instance, navigate to `Home > Alerting > Alert Rules`. Hit the '+ New alert rule' button to create an alert rule.

Here, there are a few steps:

### Naming the Rule

Enter a name for your alert rule. I named my rule 'pihole-status'.

### Defining the Query and Condition
Now, we need to define the query and the condition on which an alert will fire.

The metric I chose to query was `pihole_status`, and the label filter is set to `hostname = 192.168.103.5` which is my Pi-hole instance. When content blocking is enabled on the Pi-hole, `pihole_status` will return a 1. So, we will configure the alert condition as 'when query is equal to 0'.

![Desktop View](/assets/img/query-and-alert-condition.png)
_The query and alert condition for checking the content blocking status._


### Add Folder and Labels
Step three is to create a folder for the alert to be stored in. I created a new folder named 'pihole-alerts', and didn't configure a label.

### Defining the Alert Evaluation Behavior

Next, we need to define the alert evaluation behavior. We can create an evaluation group that will set the behavior for all of its members.

There are two properties we can configure here; 'Pending period' and 'Keep firing for'. 'Pending period' is the period during which the threshold condition must be met to configure an alert, and 'Keep firing for' is the period for which the alert will continue to show up as firing even though the threshold condition is no longer breached. I set both properties to zero seconds.

With this evaluation behavior, an example alerting flow looks like this:

1. Pi-hole content blocking is disabled 
2. Grafana evaluates the query
3. Zero seconds pass, and the alert fires
4. A 'firing' message is sent to discord.
5. Some time later, Pi-hole content blocking is re-enabled 
6. Grafana evaluates the query
7. Zero seconds pass, and the alert stops firing
8. A 'resolved' message is sent to discord.

![Desktop View](/assets/img/alert-evaluation-behavior.png)
_Alert evaluation behavior settings_

### Configuring notifications

Now we can configure the recipient of the notification, and optionally, other settings like muting alerts during a certain time period.

Since we already created a contact point, all we need to do is select it.

### Configuring the notification message

Here, we can add more content to the alert notification such as an optional summary and description of the alert, custom annotations, and a link to a dashboard.

![Desktop View](/assets/img/configuring-notification-message.png)
_Configuration for the notification message_

Now we can save the alert rule and finally test it.

![Desktop View](/assets/img/grafana-alert-rules-page.png)
_Grafana Alert Rules page, showing the new alert rule and it's health status._

## Testing the Alert Rule

To test the alert rule, I navigate to the Pi-hole server and disable content blocking.

![Desktop View](/assets/img/pihole-blocking-disabled.png)
_Content blocking is disabled_

After a few seconds, Grafana evaluates the query in the alert rule, the alert fires, and a message is routed to the contact point.

![Desktop View](/assets/img/discord-alert-firing.png)
_The alert successfully sent a message to the Discord channel._

Now, I will re-enable content blocking on the Pi-hole.

![Desktop View](/assets/img/pihole-blocking-enabled.png)
_Content blocking is enabled_

After a few seconds, another message is sent to the Discord showing that the firing alert has been resolved.

![Desktop View](/assets/img/discord-alert-resolved.png)

# Summary

In this post, I used Discord and Grafana to set up alert notifications for services in my homelab. An app and webhook link was created on Discord and set as a contact point on my Grafana instance. Then, I set up an alert rule to 



