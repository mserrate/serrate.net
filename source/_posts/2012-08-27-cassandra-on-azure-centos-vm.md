---
title: Cassandra on Azure CentOS VM
date: 2012-08-27T11:36:02+00:00
banner: /files/2012/08/cassandravm.png
categories:
  - NoSQL
tags:
  - Azure
  - BigData
  - Cassandra
  - NoSQL
---
Having some fun with Cassandra lately I wanted to figure out how to setup a working environment on the new Windows Azure VM roles, so I decided to give a try and install a **Cassandra cluster on CentOS**.

Although it’s on Ubuntu, the following article is a good guide that helped me to configure a Linux cluster: <a href="https://www.windowsazure.com/en-us/manage/linux/other-resources/how-to-run-cassandra-with-linux/" target="_blank">https://www.windowsazure.com/en-us/manage/linux/other-resources/how-to-run-cassandra-with-linux/</a>

We create the 1<sup>st</sup> VM assigning a pem certificate in order to get access by ssh:

{% img border /files/2012/08/1.png 500 340 'Create VM' %}

<!--more-->
and then we assign the DNS name:

{% img border /files/2012/08/2.png 500 232 'Assign DNS name' %}

We will connect the remaining VMs to the 1<sup>st</sup> one on the following step:

{% img border /files/2012/08/3.png 500 215 'Connect existing VM' %}

## Cassandra installation

I’ve installed Cassandra from **DataStax** source because I found that packages and documentation are pretty good, but you can install from the Apache repository as well.

Follow the next link for detailed instructions <http://www.datastax.com/docs/1.1/install/install_rpm> (just make sure to install Oracle JRE because CentOS comes with OpenJDK installed by default, and then use the _alternatives_ command to make Oracle JRE the default one). You will need to open the **9160** TPC public port on all the VM&#8217;s in the cluster (so it will be load balanced).

{% img border /files/2012/08/4.png 500 107 'Cassandra port' %}

Also, you can install **OpsCenter** (a management and monitoring web UI tool) by following <http://www.datastax.com/docs/opscenter/install/install_rhel>. Then, you will need to open the OpsCenter port:

{% img border /files/2012/08/5.png 350 252 'OpsCenter endpoint' %}

It looks like this:

{% img border /files/2012/08/7.png 500 250 'OpsCenter UI' %}

And the Cassandra&#8217;s **ring view**:

{% img border /files/2012/08/8.png 300 292 'Ring view' %}

In next posts I will cover the use of Cassandra from .NET.