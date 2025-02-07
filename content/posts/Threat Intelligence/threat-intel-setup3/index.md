---
title: "Threat Intel Setup Part 3 (Grafana)"
date: 2025-02-06T10:50:00-08:00
hero: hero.jpg
description: In this final installment of our series, we'll explore the updated Grafana dashboard specifically designed for this project. You can download and customize the dashboard to fit your needs once the data sources have been connected for each index in your Graylog instance.
menu:
  sidebar:
    name: Threat Intel Setup Part 3
    identifier: threat-and-intel-part3
    weight: 23
    parent: Purple-Team
tags: ["Grafana", "Multi-lingual"]
categories: ["Purple-Team"]
---

We've reached the culmination of our Threat Intel series for now... Throughout this journey, we've set up various systems—PFSense, Graylog, and now Grafana—to recreate a robust environment similiar to a Security Operations Center (SOC) right in your own home lab using open sourced software. Our ultimate goal is to build an ecosystem where you can seamlessly analyze network traffic and visualize data from ingested logs as we look to deploy more services in our lab.

In this post, we’ll explore how to import a dashboard I've updated overtime to quickly summarize firewall and IDS events for a given time period. I’d like to express my gratitude to [jenniferhatches](https://grafana.com/orgs/jenniferhatches) for initially creating the dashboard in 2020 using Elasticsearch as a data source. Since then, Elasticsearch has undergone a licensing change, leading AWS to fork Elasticsearch and create its own version of the software, called OpenSearch. OpenSearch is a community-driven, open-source project that is fully compatible with Elasticsearch.

If you have additional ideas on how to visualize PFSense and/or Suricata logs, please comment below. Let’s collaborate to enhance our insights!

# Grafana Install 
https://grafana.com/docs/grafana/latest/setup-grafana/installation/debian/

I'm using Grafana V10.2.3, released in 2023, which is when I first started exploring threat intelligence. The latest version may have a different UI or settings rearranged, but the principles for setting up new data sources and connecting to a dashboard remain the same. Follow the steps outlined in the link above to deploy an instance of Grafana OSS. I chose to install Grafana on the same host as Graylog, allowing us to configure a new connection for an OpenSearch index in Server Access mode, thus bypassing potential Cross-Origin Resource Sharing (CORS) requirements.

Ensure to [start the service](https://grafana.com/docs/grafana/latest/setup-grafana/start-restart-grafana/#linux) after install and check for any errors. 

{{< img src="grafana-service-status.png" height="200" width="900" align="center" title="Grafana Service Status" >}}

Once installed, you can then use the [following information](https://grafana.com/docs/grafana/latest/setup-grafana/sign-in-to-grafana/) for your initial login and please change the default admin password after login as a best practice.

{{< img src="change-pw.png" height="300" width="300" align="center" title="Change PW" >}}
# Data Source Configuration
The first step in configuring Grafana is to create new data sources, allowing Grafana to locate logs and perform queries against them.

{{< img src="new-connection1.png" height="300" width="250" align="center" title="New Connection" >}}

When adding a new connection, search for OpenSearch, which will require installation of the plugin. In my case, I already have the plugin installed:

{{< img src="Grafana-2-Opensearch-Install.png" height="150" width="900" align="center" title="OpenSearch Plugin Install" >}}

Next, we need to refer to each stream we created previously in Graylog. By viewing any message within each stream, we can save the "Stored in index" value, which will then be used as part of the OpenSearch connection details when entering the Index Name. Note that I've included an asterisk (*) at the end of the Index Name to act as a wildcard, since a new index is created every 40 days as part of the log rotation process we established earlier.

{{< img src="Grafana-3-index.png" height="450" width="900" align="center" title="Stored Index Prefix" >}}
<BR>
{{< img src="Grafana-4-data-source.png" height="650" width="600" align="center" title="OpenSearch Data Source Config Filterlog" >}}
<BR>
{{< img src="Grafana-4.2-data-source.png" height="200" width="800" align="center" title="OpenSearch Data Source Config Suricata" >}}

Save and test the connection once OpenSearch details are completed.
{{< img src="Grafana-5-test-link.png" height="250" width="900" align="center" title="Save Connection" >}}
<BR>
{{< img src="Grafana-4.3-data-source.png" height="350" width="800" align="center" title="Output Example" >}}

# Dashboard Time
You can use either link below to download the dashboard into Grafana for usage:

1. [Grafana Site](https://grafana.com/grafana/dashboards/22780-pfsense-firewall-and-ids-dashboard-2025/)
2. [My Github Repo](https://github.com/gnarly-cl0s/Threat-Intel/blob/main/Grafana-Dashboards/PFsense%20Firewall%20and%20IDS%20Dashboard%202025-1738204003974.json)
{{< img src="dashboard-import1.png" height="250" width="800" align="center" title="Import Dashboard1" >}}

{{< img src="dashboard-import2.png" height="750" width="700" align="center" title="Import Dashboard2" >}}

During the import process, you'll be presented with options such as naming the new dashboard, selecting a folder to store it in, and assigning a unique ID if it isn't already unique in your setup. Lastly, for Threat Intel Logs, we will point to the Pfsense-Filter-Logs as a starting point.
{{< img src="dashboard-import3.png" height="750" width="700" align="center" title="Import Dashboard Config" >}}

Once everything is successfully configured and imported, the final result should be a populated dashboard. On the left side, you'll see metrics from PFSense firewall rules, while the right side displays metrics for Suricata triggered alerts. Filters are located at the top of the dashboard, allowing you to drill down results based on interesting source IPs or interfaces configured in your PFSense setup. 

{{< img src="hero.jpg" height="550" width="1300" align="center" title="Grafana Dashboard" >}}
