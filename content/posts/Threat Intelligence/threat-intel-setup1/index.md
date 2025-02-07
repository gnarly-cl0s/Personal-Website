---
title: "Threat Intel Setup Part 1 (Graylog & PFSense)"
date: 2025-02-06T08:00:00-08:00
hero: hero.jpg
description: This guide is the first in the series on how to push logs from pfSense (an open-source firewall) into Graylog (an open-source log aggregator and parser). Once the logs are ingested, Graylog pipelines will be used to enrich the firewall data, which will then be visualized in a Grafana dashboard for insightful log metrics. 
menu:
  sidebar:
    name: Threat Intel Setup Part 1
    identifier: threat-and-intel
    weight: 21
    parent: Purple-Team
tags: ["Graylog", "PFSense", "Multi-lingual"]
categories: ["Purple-Team"]
---

In the ever-evolving landscape of cybersecurity, knowing what’s happening inside your network is crucial. Imagine being able to gain real-time insights into every action—what’s allowed, what’s blocked, and who’s trying to sneak in. 

This guide takes you through the process of setting up a powerful threat intelligence solution by pushing logs from pfSense (an open-source firewall) into Graylog (an open-source log aggregator and parser). By the end of this project, you’ll have not only a functional setup but also a deeper understanding of your network’s security posture. Think of it as a detective tool that helps you monitor, analyze, and respond to network threats more effectively—all while learning and having fun! Dive into the world of threat intelligence and take control of your network's security today. 

I chose PFSense for my home network firewall because it's open source and was recommended during my research on firewalls. I had some servers I was able to repurpose from work since they often scrap old, functional equipment when they receive new ones.   

These instructions outline one method for sending data from pfSense and Suricata (tested on pfSense 2.7.2) into Graylog (tested on version 5.2.2). Additionally, steps will be provided for processing Suricata logs using syslog-ng.

With a little more effort, I am confident the same approach discussed here can be applied to something like OPNSense. (I am considering transitioning to this at some point to try something new.)

I'd like to give a shout out to [Jake Stride](https://jakestride.com/) for his PFSense and Graylog series that inspired me to set this up in my home lab and make updates along the way.

# 1. Install Graylog
Instead of duplicating well-maintained installation guides, you should be able to follow the [instructions from Graylog](https://go2docs.graylog.org/current/downloading_and_installing_graylog/installing_graylog.html) to get a basic installation up and running. 
I recommend pursuing an [Ubuntu installation](https://go2docs.graylog.org/current/downloading_and_installing_graylog/ubuntu_installation.htm) to stay in sync with my setup.

Once Graylog is installed, then follow on to the next step.

# 2. Configuring Graylog
Create a new Syslog-UDP input for pfSense. Under System tab -> Inputs
Select Syslog-UDP for input type and then launch new input to provide additional details.

{{< img src="graylog1.png" height="200" width="900" align="center" title="Graylog Settings 1" >}}
<BR>
{{< img src="graylog2.png" height="200" width="900" align="center" title="Graylog Settings 2" >}}

The following configuration allows us to collect PFSense logs, as well as any other syslog messages from servers that can forward syslog to this interface. You can use any port you prefer, as long as it doesn't conflict with another service in your Graylog instance or under 1000 port range. You can set Bind address 0.0.0.0 to listen on all possible interfaces or give it the Static/DHCP IP of your Graylog instance. I bumped up the worker threads to 6 since I have other servers reaching this interface for central log management.

{{< img src="graylog3.png" height="600" width="500" align="center" title="Input Config 5" >}}

Ensure the "Expand structured data?" checkbox is enabled/toggled on to help with message parsing.
{{< img src="graylog4.png" height="300" width="500" align="center" title="Input Config 6" >}}

# 3. PFSense Logging Configuration 
With the Graylog server setup, we will now turn to the PFSense logging configuration to forward all possible syslog contents/events to the new interface we created.

From your pfSense firewall visit Status -> System Logs -> Settings. 
{{< img src="pfsense1.png" height="500" width="500" align="center" title="PFSense Settings 1" >}}

From here send the logs to Graylog by replacing the Remote log server input field with the hostname or IP address of your Graylog Server and append:5143 for port unless you decide to digress from the instructions and use a different port. Then hit Save.
{{< img src="pfsense2.png" height="500" width="500" align="center" title="PFSense Settings 2" >}}

# 4. Surricata Logging Configuration 
In my case, I have Suricata running on my WAN interface for learning purposes. However, if you were to implement something like this in your environment, you'd ideally want it running and actively blocking in the LAN, especially if you're aiming for a more stringent security setup. Typically, no malicious traffic should originate from the LAN unless there are unusual traffic patterns, suspicious protocols, or connection attempts being made to known bad hosts. Suricata provides threat hunters with valuable insights into where malicious traffic is originating from or trying to reach if a ruleset is triggered.

At first, I thought PFSense remote logging settings would suffice for sending Suricata messages to Graylog until I started reviewing incoming messages. I realized parsing Suricata logs would require a different approach, as after several trial runs, I noticed that the Suricata alerts I was expecting were not appearing. This led me to discover the FreeBSD syslog daemon will automatically truncate exported messages to 480 bytes max. Fortunately, we can install syslog-ng to configure a custom syslog process, enabling us to direct PFSense to forward any logs to our Graylog instance for further processing. 

To install Syslog-ng we go to: System -> Package -> Manager -> Available Packages and then search/download syslog-ng.
With a successful install, Syslog-ng should be available as a service in Services -> Syslog-ng.
{{< img src="syslog-install1.png" height="300" width="600" align="center" title="Syslog Install 1" >}}
<BR>
{{< img src="syslog-install2.png" height="300" width="500" align="center" title="Syslog Install 2" >}}
<BR>
{{< img src="syslog-install3.png" height="300" width="500" align="center" title="Syslog Install 3" >}}
<BR>
{{< img src="pfsense6.png" height="500" width="150" align="center" title="PFSense Settings 2" >}}

For this next part, you'll want to SSH if you have the service enabled/exposed on your PFSense host or use the hypervisor console to access the host which is the method I'll be using since I have PFSense hosted in my ESXi server. 
{{< img src="pfsense7.png" height="300" width="500" align="center" title="PFSense Settings 3" >}}
We will use this access to go to /var/log/suricata to check what directory Suricata stores alerts and log files. We will only parse Suricata alerts with this configuration, but this setup can be altered to forward other logs such as EVE or TLS and HTTP since the source script below uses a wildcard pattern to find any files in sub-directories called alerts.log. This can be changed to something like eve.json which has everything in one file, but requires parsing configuration changes I haven't worked on yet but may plan to in the future. 

{{< img src="pfsense8.png" height="200" width="800" align="center" title="PFSense Settings 4" >}}
<BR>
{{< img src="pfsense10.png" height="200" width="800" align="center" title="PFSense Settings 5" >}}

With this info we will setup and enable syslog-ng to use above path as a default log directory. I have my Graylog server in LAN so I only need syslog-ng to listen and work within LAN.
{{< img src="pfsense9.png" height="750" width="500" align="center" title="PFSense Settings 6" >}}

The most basic syslog-ng configuration has 3 components: a source, a destination, and a log object type that instructs syslog-ng to send source X to destination Y. Configuring specific logging settings is done under the Advanced tab. Click Add to start writing a config. The object names with Suricata are what we will be creating.
{{< img src="syslog-config1.png" height="300" width="550" align="center" title="Syslog config 1" >}}

The wildcard-file() source below collects log messages from multiple plain-text files and from multiple directories. The wildcard-file() source is available in syslog-ng OSE version 3.10 and later. Set the Following:
- Object Type: Source
- Object Name: Suricata

```json
{
  wildcard-file(
    base-dir("/var/log/suricata/suricata_em056333")
    filename-pattern("alerts.log")
    recursive(yes)
    follow-freq(1)
    flags(no-parse)
  );
};
```
<BR>
The log object type connects the various building blocks of Syslog-ng and thus defines the route of incoming log messages. It can contain sources, destinations, filters, flags, and other objects.  
<BR>

Add another object:
- Object Type: Log
- Object Name: Suricata

```json
{
    source(Suricata);
    destination(Suricata);
};
```
<BR>

Add the final object
- Object Type: Destination
- Object Name: Suricata

```json
{
    udp("SERVER_IP" port(5143));
};
```
<BR>
From your pfSense firewall visit Services -> Suricata -> Interfaces.
{{< img src="pfsense3.png" height="300" width="700" align="center" title="Suricata Settings 1" >}}
<BR>
Click on the pencil icon highlighted above in order to configure log settings. The areas highlighted in red are what you should pay attention to since by default EVE JSON logging is disabled and I have it set up for verbose logging of traffic I care about but you can refine as you wish. Please ensure to disable "Send Alerts to System Log" to avoid the mistake I made when initially trying to set this up.

{{< img src="Suricata-settings1.png" height="500" width="500" align="center" title="Suricata Settings 2" >}}
<BR>
{{< img src="pfsense4.png" height="500" width="500" align="center" title="Suricata Settings 3" >}}
<BR>
{{< img src="pfsense5.png" height="500" width="800" align="center" title="Suricata Settings 4" >}}

# 5. Verifying Setup
Under the System tab, navigate to Inputs within Graylog. The throughput and metrics for Syslog-UDP on port 5143 should begin to populate. By clicking Show Received Messages, you will be able to view pfSense firewall alerts and system event logs as well as Suricata logs. In the next installment of this series, we’ll explore how to route incoming messages based on log sources into streams and then we'll enhance the data by enriching it with geolocation data.

{{< img src="verify.png" height="250" width="850" align="center" title="Verify Logs Received 1" >}}
<BR>
{{< img src="verify2.png" height="100" width="1500" align="center" title="Verify Logs Received 2" >}}
