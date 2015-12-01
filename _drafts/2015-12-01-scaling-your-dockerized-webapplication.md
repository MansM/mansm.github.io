---
layout: post
title:  "Scaling your dockerized webapplication"
categories: docker consul
commentIssueId: 1
---

In this article I will show you how you can build an highly scalable environment for your webapps

We are using the following tools:

- Docker (get the docker toolbox: https://www.docker.com/docker-toolbox)
- Consul
- Haproxy
- Apache with php
- Git repo with example code: https://github.com/MansM/docker-scalingdemo

You only need to install the docker toolbox, the other tools are in docker containers.

##Use case:##
You have an webapplication that became quite popular. You want to dynamically scale up and down by adding containers.
But you dont want to update manually your haproxy configuration

##Tooling:##

- Docker: In the current IT environment there is never enough speed, VM's are slow and containers are hot. Docker is currently the most popular container software.
- Haproxy: Opensource load balancer for TCP and HTTP based services
- Consul: A tool for service discovery and configuration

###Apache with php:###
Just an example of an webapplication, but the you can use any other webapplication you like

##Docker containers
As mentioned before I am using docker containers in this setup. Three different containers can be identified:

- Consul server
- Haproxy with consul-template
- Webservice with consul agent

The consul server is the consul-server container made by gliderlabs, to start it:
{% highlight bash %}
docker run --rm -p 8500:8500 gliderlabs/consul-server -bootstrap
{% endhighlight %}

to view the consul server ui, you can browse to the docker ip:8500, you can locate the ip with running the command:
{% highlight bash %}
docker-machine ip default
{% endhighlight %}
![Consul ui](/images/consul-ui.png)

All the containers are listed in the docker-compose.yml. This file is used by docker-compose to run the multi-container environment.



##sources:##
http://sirile.github.io/2015/05/18/using-haproxy-and-consul-for-dynamic-service-discovery-on-docker.html
http://technologyconversations.com/2015/07/02/scaling-to-infinity-with-docker-swarm-docker-compose-and-consul-part-14-a-taste-of-what-is-to-come/