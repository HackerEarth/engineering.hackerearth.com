---
layout: post
title: "Monitoring and alert system using Graphite and Cabot"
description: "Setting up an alert and monitoring system for a product using Graphite and Django"
category:
tags: [Monitoring, Alerting, Oncall, Graphite, Cabot, InfluxDB, OpenSource, DevOps]
---

## Introduction ##

The infrastructure that powers a product and all of the services that it provides can be huge and complex because the product is scaled to serve millions of users.
In most cases, each service might depend on various components for seamless functioning. With a product that houses a variety of features with critical infrastructure components and services powering these features,
it becomes vital to monitor these components and services and keep them running at any cost. 

This monitoring system has to handle the following:

  1. Gathering data from all the components and services
  2. Storing the data efficiently and in an easily accessible manner
  3. Visualizing the data for faster comprehension
  4. Making sense of this data and relaying alerts to the respective owners of the services and components
  5. Managing the on call team and alerting them immediately

## Components of this system ##

This system consists of the following components that work together,

  - **Collection** : Tool for collecting metrics from all the infrastructure components
  - **Forwarding** : Tool for aggregating the metrics that are recieved various machines and services and routing it to different storage backends
  - **Visualization** : Tool for generating graphs and visualizing metrics
  - **APIs** - Backend that provides APIs to query the metrics data for dashboards and alerting systems
  - **Storage backends** : Database for storing the time-series metrics data
  - **Monitoring** : Monitoring system for performing checks on the metrics and sending alert notifications

There are a lot of tools out there for each of these components that can be put together to build this system rather than writing it from the scratch.
We were very happy with what [Cabot](http://cabotapp.com/) had to offer as a monitoring and alert system. Since it used Graphite APIs for the monitoring checks,
we decided to use Graphite to record and serve all our metrics.

***
## Traditional approach ##

Graphite is usually used:
  - Carbon as a forwarding/relaying component
  - Graphite-Web as a web and API component
  - Whisper as a library for storing time-series data with MySQL or PostgreSQL as the database
  - Grafana for visualization and any tool like collectd or statsd as a metric collector

### Limitations ###

While these are useful, there are many issues that surface as you try to scale this system, 
Some of the major limitations seen with this approach include,
  - Carbon is written in Python, and cannot scale to millions of metrics that are being reported to it. It starts dropping metrics when it is unable handle the load
  - Using databases like MySQL and PostgreSQL for storing time-series data is not efficient. Graphs on Grafana start rendering very slowly. Its understandable as these databases were not built for time-series data


<img src="/images/cabot-1.png" alt="Usual Approach" style="width: 70%;display: block;margin: 0 auto;"/>


## Influx to the rescue! ##

InfluxDB is a database that is written in Go, primarily for time-series metric data. This database is by far the best option for a database to be used with Graphite. We also found alternatives to other components to use InfluxDB with Graphite.

Some of the alternative components that we found to be better include,

  - **[carbon-relay-ng](https://github.com/graphite-ng/carbon-relay-ng)**: This is a carbon that is written in Go. It can scale to 2500 metrics/s or 150k minutely. It is used for forwarding the metrics to InfluxDB which has a graphite listener to receive the data in the Graphite format
  - **[Grafana](https://grafana.com/)**: This has the capability of adding graphite as a data source and helps in creating and rendering graphs effortlessly
  - **[graphite-api](https://github.com/brutasse/graphite-api)**: This is the API component. It is the APIs in graphite-web without the web interface and is much lighter. It uses [InfluxGraph](https://github.com/InfluxGraph/influxgraph) as a storage backend.
  - **[InfluxDB](https://www.influxdata.com/)**: serves as a kickass backend to store the metrics data!


<img src="/images/cabot-2.png" alt="Better Approach" style="width: 70%;display: block;margin: 0 auto;"/>

***
## Putting these components together ##

### Carbon Relay Ng ###

You can install carbon-relay-ng from [here](https://packagecloud.io/raintank/raintank/packages/ubuntu/trusty/carbon-relay-ng_0.8-1_amd64.deb). The following points have to be kept in mind while using it:

  - The conf file in ***/etc/carbon-relay-ng/carbon-relay-ng.conf*** has the
    params ***spool-dir*** and ***pid-file***. In case of errors due to these parameters, set them to ***/var/spool/carbon-relay-ng*** and ***/var/run/carbon-relay-ng.pid***. Create the directory in spool, if required.
  - It supports routing to various sources by creating routes in the ***config file***. For more information, read the documentation.
  - It allows the blacklisting of metrics based on the prefix, regex, or substring match.
  - It provides aggregators to batch metrics for an interval and appies aggregation functions on the values. The aggregated value is then routed to the destination.
  - It also allows rewriters to rewrite metric names.
  - It comes with a UI where new routes can be added during runtime.

#### Sample Route ####

```
Format : addRoute <type> <key> [opts]   <dest>  [<dest>[...]]
'addRoute sendAllMatch collectd  127.0.0.1:2005 prefix=hemetrics'
```

Multiple such routes can be added to the ***init*** list in the config file. Here the command **addRoute** adds a new route of the type **sendAllMatch**, **which sends metrics to all destinations that match!**. Metrics can be matched by using prefixes, substring, or regex matches.

In this case the route has the key ***collectd*** and sends metric data with the prefix ***hemetrics*** to all matching destinations where the destination is expected to be a TCP endpoint (IP:port combination).

[**carbon-relay-ng**](https://github.com/graphite-ng/carbon-relay-ng#tcp-interface) provides a lot more options for matching and transforming data. For more information, read the documentation. :)

***
### Collectd ###

Collectd is used for the metrics-collection component. You can get ***collectd*** from [here](https://collectd.org/download.shtml). ***Collectd*** provides a lot of [plugins](https://collectd.org/documentation/manpages/collectd.conf.5.shtml) for collecting various metrics.
Some important plugins to look at are CPU, memory, and disk. It also provides plugins for other tools like MySQL, Redis, PostgreSQL and amqp.

Custom metrics can be reported to ***carbon-relay-ng*** using the Python plugin that runs a Python script to collect metrics.
This is an [example](https://github.com/chrisboulton/collectd-python-mysql/blob/master/mysql.py) of writing custom-service plugins (in Python) that are used by ***collectd***. Most importantly the write_graphite plugin is used to report metrics to ***carbon-relay-ng*** in the graphite line format.

This is the format of the ***write_graphite*** plugin that should be added in the ***collectd*** config file
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

# Important: You must use the right prefix. If it does not match with
# any of the carbon routes that have been configured, then carbon-relay-ng will
# drop the metrics.
```

The intervals in which ***collectd*** sends metrics is configured a the global level in the config file for all the plugins by using the ***Interval*** parameter. It can be specified inside each plugin to override the interval for that plugin.

***Collectd*** is one of the tools that is used to report metrics. Many other tools can be used to report metrics to graphite with this architecture. Some alternatives can be found [here](http://graphite.readthedocs.io/en/latest/tools.html#collection).

Carbon or carbon-relay-ng in our case, can be fed externally too by using the line-text protocol. For more information, read the relevant documentation [here](http://graphite.readthedocs.io/en/latest/feeding-carbon.html).

***
### Grafana ###

Grafana as a visualization component, provides a plethora of options like histograms, heat maps, and graphs. Grafana provides alerting options and notifications on alerts to PagerDuty, Slack, and a few other services. However, since we wanted something
more than that we decided to deploy Cabot as the monitoring and alerting component.

Grafana enables us to create beautiful dashboards and renders it seamlessly for the on-call team to monitor various components in a single dashboard. Installing grafana
and creating dashboards is a cakewalk. Start creating your dashboards [here](http://docs.grafana.org/installation/debian/).


<img src="/images/cabot-3.png" alt="Usual Approach" style="width: 85%;display: block;margin: 0 auto;"/>

***
## Graphite API  & Influx Graph ##

[**Graphite-API**](https://github.com/brutasse/graphite-api) is grahpite-web with just the HTTP APIs and not the web interface. It implements most of the APIs. To make Graphite-API use InfluxDB as the datasource, install [**Influx Graph**](https://github.com/InfluxGraph/influxgraph) (storage plugin for Graphite-API).

The configuration for Graphite-API resides in ***/etc/graphite-api.yaml***. An example for it can be found [here](https://github.com/InfluxGraph/influxgraph/blob/master/graphite-api.yaml.example)

Installing Influx Graph installs Graphite-API and a lot more dependencies including InfluxDB. Graphite-API provides a lot of configuration options for using standard templates for queries, caching queries in memcache, aggregation functions, and grouping data bases on intervals combined with query-retention policies to cache queries of certain intervals for specific amounts of time.

When templates are added, the API expects queries for the metrics of the format that is specified in these templates only. Any other queries will return zero results.

Graphite-API can be deployed in various [ways](https://graphite-api.readthedocs.io/en/latest/deployment.html). At HackerEarth we have deployed it using NGINX and uWSGI.

***
## Cabot ##

Cabot is an open-source monitoring and alert system written in Python and Django. It provides the best features of PagerDuty free of cost.

The documentation is pretty good for setting it up and getting it work without any hassles. Some of the best features of Cabot include:

  - Web interface to configure services, checksi, and users
  - Good coupling of services, checks, and users. This allows certain default users to be notified for every service.
  - Checks that include graphite metric checks, jenkins job checks, and http endpoint checks.
  - Alerts sent through email, Hipchat, Slack, phone and SMS (Twilio).
  - Recovery instructions can be tagged with every check
  - Easy integration of custom alerts.
  - On-call management, users on call are picked up from a Google Calendar and updated every half hour.
  - Easy deployment

All of the cabot configuration options can be found [here](http://cabotapp.com/use/configuration.html).

**Note**: Cabot uses Celery to run all the service check tasks. The configuration parameter ***CELERY\_BROKER\_URL*** requires a redis host with the right authentication. Also set the ***WWW\_HTTP\_HOST*** to the FQDN of the server.

Check if all the checks are running by visiting /status/ endpoint.


<img src="/images/cabot-4.png" alt="Usual Approach" style="width: 85%;display: block;margin: 0 auto;"/>

<br/>

<img src="/images/cabot-5.png" alt="Usual Approach" style="width: 85%;display: block;margin: 0 auto;"/>


***

## Final Words ##

## References ##

