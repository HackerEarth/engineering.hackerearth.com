---
layout: post
title: "Aggregating Apache logs with Fluentd and Amazon S3"
description: "HackerEarth project is hosted on Amazon EC2. At any given point of
time many webservers are running concurrently serving thousands of requests."
category:
tags: [Apache, Logs, Fluentd, Amazon, S3, EC2]
---
{% include JB/setup %}

HackerEarth infrastructure is hosted on Amazon services. At any given point of time many
webservers are running concurrently serving thousands of requests. This
generates tons of access and error logs on each server separately. The task
here was to  parse the logs on all these webservers and store them at one place
in a format that can further be used to derive meaningful insights from the
data. We tried to accomplish this using fluentd and Amazon S3.

####Mechanism

Fluentd does the following things:

1. Continuosly tails apache log files.
2. Parses incoming entries into meaning fields like ip, address etc and buffers them.
3. Writes the buffered data to Amazon S3 periodically.

####Installation
The stable version of fluentd is called td-agent and we are using the same for
our purpose here.
For ubuntu 12.04 LTS the following shell command will install td-agent on your
system.

 `curl -L http://toolbelt.treasuredata.com/sh/install-ubuntu-precise-td-agent2.sh | sh`

The other supported operating systems and installation methods are listed
[here](http://docs.fluentd.org/articles/apache-to-s3).

Please note that if you are installing ruby using Ruby Gems, you will have to
install Amazon S3 output plugin separately. This can be done by:

 `gem install fluent-plugin-s3`

####Configuration

Once td-agent is installed you will find a td-agent.conf file in /etc/td-agent/
directory. For parsing apache access logs you will need to add the following
configuration:

    <source>
        type tail                           # for continuosly tailing the log
        format apache2                      # for default format of apache access logs
        time_format %d/%b/%Y:%H:%M:%S %z    # time format in access logs
        path /var/log/apache2/access.log    # path from where log is to be read

        # if td-agent restarts, it starts reading from the
        #last position td-agent read before the restart
        pos_file /var/log/td-agent/apache2.access_log.pos

        tag s3.apache.access                # for identifying the log stream uniquely
    </source>

    <match s3.*.*>
        type s3                             # plugin for writing the log to s3
        aws_key_id <YOUR AWS KEY ID>
        aws_sec_key <YOUR AWS SECRET KEY>
        s3_bucket <YOUR S3 BUCKET NAME>
        path <PATH ON BUCKET>

        #place where the stream is stored before being written on s3
        buffer_path /var/log/td-agent/s3/

        # this specifies the interval at which logs are to written to s3.
        # this format specifies daily writes
        time_slice_format %Y%m%d

        # the amount of time fluentd will wait for old logs to arrive

        time_slice_wait 10m

        buffer_chunk_limit 256m            # max size of a buffer chunk
    </match> 

This configuration is for the default apache access.log file and the filter for
this is predefined in td-agent(i.e format apache2). If you want to use some
other log format you will need to write a regular expression for parsing those
logs.

If you are using vhost_combined format for access logs, all you need to do is to replace
apache2 in the second line the source block with this:

    format /^(?<virtualhost>[^ ]*)[:](?<port>[^ ]*) (?<host>[^]*)"(?<forwardedfor>[^\"]*)" [^ ]* (?<user>[^ ]*)\[(?<time>[^\]]*)\]"(?<method>\S+)(?: +(?<path>[^ ]*) +\S*)?" (?<code>[^ ]*)(?<size>[^ ]*)(?:"(?<referer>[^\"]*)" "(?<agent>[^\"]*)")?$/

Please refer to this [article](http://docs.fluentd.org/articles/in_tail) for other pre-available log formats.

####Testing

To test your configuration run this command in your terminal

 `ab -n 100 -c 10 http://localhost/`

Now login to your [Amazon console](https://console.aws.amazon.com) and check
the generated logs. With the above configuration you should be sucessfully able
to write your apache access logs to Amazon S3 on a daily basis.

Further these logs can be used to analyzed using ElasticSearch and
LogStash/Kibana to analyze all the requests that your web servers receive.

This blog is mostly a reproduction of the official [fluentd
blog](http://docs.fluentd.org/articles/apache-to-s3) with a little detailed
expanation. 

P.S. I am a developer at [HackerEarth](http://www.hackerearth.com)
Reach out to me at
virendra@hackerearth.com for any suggestion, bugs or even if you just want to
chat! Follow me [@virendra2334](https://twitter.com/virendra2334)

*Posted by [Virendra Jain](http://www.hackerearth.com/users/virendra/)*
