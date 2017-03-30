---
layout: post
title: "Monitoring and Alert system using Graphite and Cabot"
description: "Setting up an alert and monitoring system for a product using Graphite and Django"
category:
tags: [Monitoring, Alerting, Oncall, Graphite, Cabot, InfluxDB, OpenSource, DevOps]
---

## Introduction ##

The infrastructure that powers a product and all of the services it provides can be tremendously huge and complex as the product is scaled to serve millions of users.
In most case each service might depend of various components for seamless funtioning. With such a product which houses a variety of features with critical infrastructure components and services powering these features,
it becomes vital to monitor these components and services and keep them running at any cost. 

This monitoring system has to handle everything from gathering data from all the components and services, storing it efficiently and in an easily accessible manner, visualizing the data for faster comprehension,
making sense out of this data and relaying alerts to the respective owners of the services and components to lastly managing the on call team and alerting them immediately.

This system consists of the following components that work togather,

  - **Collection** - A tool to collect metrics from all infrastructure components.
  - **Forwarding** - A tool for aggregating the metrics coming from various machines and services and routing it to different storage backends.
  - **Visualization** - A tool to generate graphs and visualize these metrics.
  - **APIs** - A backend that provides APIs to query the metrics data for dashboards and alerting systems.
  - **Storage Backends** - A database to store the time-series metrics data.
  - **Monitoring** - A monitoring system that performs checks on the metrics and send alert notifications.

There are a lot of tools out there for each of these components that can be put togather to build this system rather than writing it from the scratch.
We were very happy with what [Cabot](http://cabotapp.com/) had to offer as a monitoring and alert system. Since it used Graphite APIs for the monitoring checks,
we decided to use Graphite to record and serve all our metrics.

***
## The Usual Approach ##

Graphite is usually used with carbon as a forwarding/relaying component, graphite-web as a web and API component, whisper as a library for storing time-series data with MySQL or PostgreSQL as the database, 
grafana for visualization and any tool like collectd or statsd as a metric collector. But there are many issues that surface as we try to scale this system.

Some of the major limitations seen with this approach are,

  - carbon is written in Python, and cant scale to millions of metrics being reported to it. It starts dropping metrics when it cant handle the load
  - Using databases like MySQL and PostgreSQL for storing time-series data isnt efficient. Graphs on Grafana start rendering very slowly. Its understandable as these databases were not built for time-series data.


<img src="/images/cabot-1.png" alt="Usual Approach" style="width: 70%;display: block;margin: 0 auto;"/>


## Influx to the rescue! ##

InfluxDB a database written in Go, primariy for time-series metric data is the by far the best option for a database with Graphite. We also found alternatives to other components to use InfluxDB with Graphite.

Some of the alternative components we found to be better are,

  - **[carbon-relay-ng](https://github.com/graphite-ng/carbon-relay-ng)** is a carbon written in Go that can scale to 2500 metrics/s or 150k minutely which was used for forwarding the metrics to InfluxDB which has a graphite listener to recieve the data in the Graphite format
  - **[Grafana](https://grafana.com/)** has support to add graphite as a data source and helps create and render graphs effortlessly
  - **[graphite-api](https://github.com/brutasse/graphite-api)** for the APIs component, this is the APIs in graphite-web without the web interface. It is much lighter. It uses [InfluxGraph](https://github.com/InfluxGraph/influxgraph) as a storage backend.
  - **[InfluxDB](https://www.influxdata.com/)** serves as a kickass backend to store the metrics data!


<img src="/images/cabot-2.png" alt="Better Approach" style="width: 70%;display: block;margin: 0 auto;"/>

Going ahead, lets see in detail how these components can be put together.

***
## Carbon Relay Ng ##

carbon-relay-ng can be installed from [here](https://packagecloud.io/raintank/raintank/packages/ubuntu/trusty/carbon-relay-ng_0.8-1_amd64.deb). Some of the points to be noted while using it are,

  - The conf file in /etc/carbon-relay-ng/carbon-relay-ng.conf has the params spool-dir and pid-file. In case of errors due to these parameters, set them to '/var/spool/carbon-relay-ng' and '/var/run/carbon-relay-ng.pid'. Create the directory in spool if needed.
  - It supports routing to various sources by creating routes in the config file. Read the documentation for more information.
  - It allows to black list metrics based on the prefix, regex or substring match.
  - It provides aggregators to batch metrics for an interval and apply aggregation functions on the values. The aggregated value is then routed to the destination.
  - It also provides rewriters to rewrite the metric names.
  - It comes with a UI where new routes can be added during runtime.

A sample route will look like,

```
Format : addRoute <type> <key> [opts]   <dest>  [<dest>[...]]
'addRoute sendAllMatch collectd  127.0.0.1:2005 prefix=hemetrics'
```

Multiple such routes can be added to the init list in the config file. Here the command **addRoute** adds a new route of type **sendAllMatch**(***sends metrics to all destinations that match***). Metrics can be matched using prefixes, substring or regex matches.
In this case the route has key 'collectd' and sends metric data with prefix 'hemetrics' to all matching destinations where the destination are expected to be a TCP endpoint(ip:port combination).

[**carbon-relay-ng**](https://github.com/graphite-ng/carbon-relay-ng#tcp-interface) provides a lot more options for matching and transforming data. Read the docs for more details. :)

***
## Collectd ##

We use collectd for the metrics collection component. You can get collectd from [here](https://collectd.org/download.shtml). Collectd provides a lot of [plugins](https://collectd.org/documentation/manpages/collectd.conf.5.shtml) for collecting various metrics.
Some important plugins to look at are cpu, memory and disk. It also provides plugins for other tools like mysql, redis, postgresql and amqp. Custom metrics can be reported to carbon-relay-ng using the python plugin that runs a python script to collect metrics.
This is an [example](https://github.com/chrisboulton/collectd-python-mysql/blob/master/mysql.py) for writing custom service plugins in python used by collectd. Most importantly we use the write_graphite plugin to report metrics to carbon-relay-ng in the graphite line format.

```
<Plugin write_graphite>     
    <Node "graphite">       
        Host "<carbon-server-ip>"
        Port "<carbon-server-port>"         
        Protocol "tcp"      
        LogSendErrors true  
        Prefix "random_prefix."
        StoreRates true     
        AlwaysAppendDS false
        EscapeCharacter "_" 
    </Node>                 
</Plugin>

# Important thing to note is the prefix you are using. If it doesnt match with
# any of the carbon routes configured carbon-relay-ng will just drop it.
```

The intervals in which collectd sends metrics is configured at the global level in the config file for all the plugins using the ***Interval*** parameter. It can be specified inside each plugin to override the interval for that plugin.

Collectd is one of the tools out there to record metrics. Many other tools can be used to report metrics to graphite with this architecture. Some alternatives can be found [here](http://graphite.readthedocs.io/en/latest/tools.html#collection).

Lastly carbon can be fed externally too using the line-text protocol. Please read the documentation for it [here](http://graphite.readthedocs.io/en/latest/feeding-carbon.html).

***
## Grafana ##

Grafana as a visualization component provides a plethora of options like histograms, heatmaps and graphs. Grafana does provide alerting options and notifications on alerts to PagerDuty, Slack and a few other options. But since we wanted something
more than that we decided to deploy Cabot as the monitoring and alerting component. Grafana enables us to create beautiful dashboards and renders it seamlessly for the oncall team to monitor various components in a single dashboard. Installing grafana
and creating dashboards is a cake walk. Get started [here](http://docs.grafana.org/installation/debian/).


<img src="/images/cabot-3.png" alt="Usual Approach" style="width: 85%;display: block;margin: 0 auto;"/>

***
## Graphite API  & Influx Graph ##

[**Graphite-API**](https://github.com/brutasse/graphite-api) is Grahpite Web without the web interface and just the HTTP APIs. It implements most of the APIs. But to make it use InfluxDB as the datasource, install [**Influx Graph**](https://github.com/InfluxGraph/influxgraph) which is the storage plugin
for Graphite-API. The configuration for Graphite-API resides in /etc/graphite-api.yaml. An example for it can be found [here](https://github.com/InfluxGraph/influxgraph/blob/master/graphite-api.yaml.example)

**NOTE** : Install Influx Graph installs Graphite-API and a lot more dependencies including InfluxDB.

Graphite-API provides a lot of configuration options for using standard templates for queries, caching queries in memcache, aggregation functions and grouping data bases on intervals combined with query retention policies to cache queries of certain intervals for certain amount of time.

**NOTE** : On adding templates, the API expects queries for the metrics of the format specified in these templates only. Any other queries will return zero results.

Graphite-API can be deployed in various [ways](https://graphite-api.readthedocs.io/en/latest/deployment.html). At HackerEarth we have deployed it using Nginx and uwsgi.

***
## Cabot ##

Cabot is an open source monitoring and alert system written in Python and Django that provides the best features of PagerDuty free of cost. The documentation is pretty good to set it up and get it working withoutany hassle. Some of the best features of Cabot are,

  - A web interface to configure services, checks and users
  - Good coupling of services, checks and users. This allows certain default users to be notified for every service.
  - Checks that include graphite metric checks, jenkins job checks and http endpoint checks.
  - Alerts sent through email, Hipchat, Slack, phone and sms (Twilio)
  - Recovery instructions can be tagged with every check
  - Easy integration of custom alerts.
  - Oncall management, users oncall is picked up from a Google Calendar and is updated every half an hour.
  - Easy deployment

All of the cabot configuration options can be found [here](http://cabotapp.com/use/configuration.html).

**NOTE**: Cabot uses celery to run all the check tasks. The configuration parameter ***CELERY\_BROKER\_URL*** requires a redis host with right authentication. Also set the ***WWW\_HTTP\_HOST*** the FQDN of the server.

Lastly you can check if all the checks are running by visiting /status/ endpoint.


<img src="/images/cabot-4.png" alt="Usual Approach" style="width: 85%;display: block;margin: 0 auto;"/>

<br/>

<img src="/images/cabot-5.png" alt="Usual Approach" style="width: 85%;display: block;margin: 0 auto;"/>


***

## Final Words ##

## References ##

