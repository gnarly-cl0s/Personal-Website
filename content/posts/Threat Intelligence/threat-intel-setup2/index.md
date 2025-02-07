---
title: "Threat Intel Setup Part 2 (Graylog)"
date: 2025-02-06T09:00:00-08:00
hero: hero.jpg
description: This guide is the second in the series where additional Graylog settings will be configured to help parse incoming log messages from PFSense and Suricata. Extractors will be imported into Graylog to help parse PFSense data as well as setup service for Port Mappings and Geo IP look ups. 
menu:
  sidebar:
    name: Threat Intel Setup Part 2
    identifier: threat-and-intel-part2
    weight: 23
    parent: Purple-Team
tags: ["Graylog", "Multi-lingual"]
categories: ["Purple-Team"]
---

Welcome back to the Threat Intel series! In this chapter, we’re rolling up our sleeves and diving deeper into Graylog’s powerful features to optimize the way it handles and enriches log data from PFSense and Suricata. If you’re ready to transform raw log messages into actionable insights, this guide is for you!

# What’s on the Agenda?

We’ll configure essential Graylog settings to streamline parsing incoming log messages. By the end of this deep dive, you’ll have a setup capable of:

- Efficiently parsing PFSense logs with the help of pipeline rules.
- Setting up Geo-Location Processor for port mappings and GeoIP lookups, giving us enriched, context-aware data.
- Integrating data from Suricata for an additional layer of network visibility.

Logs are only as valuable as the insights you can extract from them. Without proper processing, critical data can remain hidden in plain sight. By leveraging Graylog’s capabilities, you’ll not only make sense of the data but also enhance it turning your home lab into a robust, threat-aware system.

# 1. Indices
First, create an Index Set for each type of syslog message you plan to retain. Each Index Set contains the necessary settings for Graylog to create, manage, and populate search backend indices. It also handles index rotation and data retention based on your specific requirements.
{{< img src="graylog5.png" height="200" width="1000" align="center" title="Graylog Settings 1" >}}

{{< img src="index-config.png" height="400" width="700" align="center" title="Index Set Config" >}}

We will create 3 index sets since we can expect
syslog messages for:
- General PFSense server events from OS (pfsense_alerts)
- Suricata Alerts (pfsense_suricata)
- PFSense Filterlog for PASS and FAIL events based on configured firewall rules (pfsense_filterlog)

Each index set is setup with 1 shard which can be expanded for better  scalability and performance when dealing with large volumes of log data. In this case 1 shard shall do since we only have 1 Graylog server and typically the recommended number of shards should be proportional to the numbers of nodes in your cluster. 

The retention for the messages is at least 30 days and then deleted after 40 days. I personally only care for a months worth of data to save on storage space but you can define this dependent upon how much space you have to store these logs that can get big quickly in chatty networks with a lot of devices. 
{{< img src="graylog6.png" height="350" width="1000" align="center" title="Graylog Settings 2" >}}

# 2. GeoIP and Service Lookup Setup
Graylog geolocation is a feature that allows users to map log data by adding geolocation information to it. Graylog ships with geolocation capabilities by default but additional configuration is still required.

To start, download a geolocation database. As of version 4.3, both MaxMind and IPInfo databases are supported by Graylog. I created an account for MaxMind and followed  [GeoIP Update program](https://dev.maxmind.com/geoip/updating-databases/?lang=en#1-install-geoip-update) since we plan to use their binary database format and this will help provide automatic updates after cron job created for this.

Log in to your [Maxmind account portal](https://www.maxmind.com/en/accounts/current/license-key/GeoIP.conf) to download a partial config file and save it in your configuration directory (e.g., /etc/) as GeoIP.conf 
The LicenseKey will need to be replaced with a License key value you generate through MaxMind website along with updating EditionIDs to include Geolite2-ASN if not there already. 
{{< img src="max-mind-license.png" height="300" width="600" align="center" title="Max Mind License" >}}
<BR>
{{< img src="geo-ip-config2.png" height="300" width="600" align="center" title="Update Geo Config" >}}
Make sure your Graylog server can make HTTPS connections to the following hostname:

`mm-prod-geoip-databases.a2649acb697e2c09b632799562c076f2.r2.cloudflarestorage.com`

Download [geoipupdate ](https://github.com/maxmind/geoipupdate/releases) into your Graylog server.
```bash
wget https://github.com/maxmind/geoipupdate/releases/download/v7.1.0/geoipupdate_7.1.0_linux_amd64.deb
sudo dpkg -i geoipupdate_7.1.0_linux_amd64.deb
```
Once downloaded, you can then run `sudo geoipupdate -v` to download latest GeoIP datasets. Datasets are stored in /usr/share/GeoIP.
{{< img src="download-example.png" height="200" width="600" align="center" title="Downloads" >}}

Since I am using Ubuntu to host my Graylog service in my homelab, we can setup a crontab for geoipupdate so that it'll run the command every day at 3 AM and store any output or errors in /var/log/geoipupdate.log. Do the following:
```bash
 sudo crontab -e
 # Enter at the bottom of root crontab
 0 3 * * * /usr/bin/geoipupdate -v > /var/log/geoipupdate.log 2>&1
```

You can then download a [CSV](https://github.com/gnarly-cl0s/Threat-Intel/blob/main/Graylog-Content/service-names-port-numbers.csv) from my Github Repo that I use for port service lookups into the following location in your Graylog server: `/etc/graylog/server/service-names-port-numbers.csv`

Ensure the following message processor order matches with your Graylog configuration settings so messages are handled in the correct order when parsing data:
{{< img src="message-processors.png" height="350" width="800" align="center" title="Message Processors" >}}

Once both respective downloads have been completed, we will activate the Geo IP DB in Graylog:
{{< img src="geo-ip-config.png" height="350" width="600" align="center" title="Geo Ip Enabled" >}}

# 3. Content Pack Import
When reviewing log messages received before using a content pack, you'll initially notice message content unparsed due to needing to create some extractors to help define the elements of the log file based on a pattern.
{{< img src="unparsed-data.png" height="350" width="700" align="center" title="Unfiltered Logs" >}}

The remaining Graylog configuration involves uploading a content pack I’ve generated for this project. This content pack mirrors my home lab setup in the following [Github repo](https://github.com/gnarly-cl0s/Threat-Intel/blob/main/Graylog-Content/Threat-Intel-Log-Parsing-Content-Pack.json). The JSON file contains preconfigured settings that will automatically create the remaining components for us, including streams, pipelines, and parsing rules.
{{< img src="content-pack1.png" height="450" width="900" align="center" title="Content Pack Settings1" >}} 

When you successfully upload the content pack, you'll see the new pack ready for installation. You can then proceed to install the pack to import the settings. 
{{< img src="content-pack2.png" height="450" width="900" align="center" title="Content Pack Settings2" >}} 

{{< img src="content-pack3.png" height="150" width="1000" align="center" title="Content Pack Installed" >}} 
<BR>
This will result into 3 new streams created along with pipeline and respective rules to parse messages. 

{{< img src="pipelines.png" height="450" width="800" align="center" title="Installed Pipelines" >}} 
<BR>
{{< img src="streams.png" height="350" width="1000" align="center" title="Installed Streams" >}} 

Notice how the Streams are currently configured with Default index set which will need to change to the indicies we created at the [start of this blog post](#1-indices) so that the log retention settings are applied to newly created streams.

{{< img src="stream-update1.png" height="350" width="500" align="center" title="Stream Index Update 1" >}} 
<BR>
{{< img src="stream-update2.png" height="350" width="1000" align="center" title="Stream Index Update 2" >}} 

At this point you can either continue to the last part of this series where we install Grafana into our Graylog instance and start looking into making dashboards with our parsed data or you can continue to read below where I provide some additional context into what the content pack contains and how everything is setup.

# 4. Content Pack Breakdown
{{< mermaid align="center" >}}
graph TD;
    A(PFsense and Suricata Syslog Messages) -->|UDP Port 5143| B(Syslog-UDP Input Interface on Graylog)
    B --> C{Streams}
    C -->|index=pfsense_alerts| D(Pipeline Alerts)
    C -->|index=pfsense_filterlog| E(Pipeline Filterlog)
    C -->|index=pfsense_suricata| F(Pipeline Suricata)
    D --> G{Stage 0 and 1} 
    E --> H{Stage 0 and 1}
    F --> I{Stage 0 and 1}
    G --> L{Rules}
    H --> M{Rules}
    I --> N{Rules}
{{< /mermaid >}}

Graylog starts by exposing inputs for various log sources. As log messages are received, Graylog then routes them into specific streams. This routing process is governed by rules associated with each stream, which define the criteria for determining which messages should be directed to which streams.
{{< img src="stream-rule.png" height="350" width="500" align="center" title="Stream Rule" >}} 
<BR>
{{< img src="stream-rule2.png" height="350" width="500" align="center" title="Stream Rule2" >}} 

Pipelines let us transform and process messages coming from streams. Pipelines consist of stages where rules are evaluated and applied. Messages can go through one or more stages. We currently have 2 stages setup for each stream:
- First stage parses out fields from log messages
- The second stage involves extracting the Source IPs from the log messages. These extracted IPs are then used to perform GeoIP lookups, with the query results being pushed back into the log message. 
{{< img src="pipelines2.png" height="350" width="500" align="center" title="Pipeline Overview" >}} 
<BR>

The first screenshot shows stage rules for PFSense general filter logs while the subsequent screenshot is for Suricata alerts.

{{< img src="stages2.png" height="450" width="700" align="center" title="PFSense Filter Stage Rules" >}} 
<BR>
{{< img src="stages1.png" height="450" width="700" align="center" title="Suricata Filter Stage Rules" >}} 

If you are curious to learn more about how to write the Rule source then refer to [Graylog documentation](https://docs.graylog.org/docs/rules) which helped me understand the rule structure and syntax to create rules through source code editor in Graylog. Check out https://pipe-dreams.vercel.app/ for some AI help!
{{< img src="rule-source.png" height="450" width="800" align="center" title="Rule Source Example" >}} 

Lookup tables are generated and employed in pipeline processing to enhance log messages by enriching them with additional details and maintaining a cache of previously queried results. This process involves appending the service name corresponding to the targeted port, as well as the city and Autonomous System Number (ASN) linked to the source IP addresses identified in the logs. These settings point back to the services lookup [CSV along with GeoIP datasets](#2-geoip-and-service-lookup-setup) we downloaded into Graylog.
{{< img src="lookup-table-setup1.png" height="250" width="800" align="center" title="Lookup Table Settings1" >}} 
<BR>
{{< img src="data-adapter.png" height="250" width="800" align="center" title="Data Adapter" >}} 
<BR>
{{< img src="caches.png" height="250" width="800" align="center" title="Cache" >}} 

This allows us to analyze and plot the frequency of messages received from the origin of potential attacks.  

{{< img src="geoip2.png" height="500" width="350" align="center" title="GeoIP2" >}} 
<BR>
{{< img src="geoip.png" height="200" width="350" align="center" title="GeoIP" >}} 

In the next and final part of this series we will learn how to pull this data into Grafana and visualalize the data.