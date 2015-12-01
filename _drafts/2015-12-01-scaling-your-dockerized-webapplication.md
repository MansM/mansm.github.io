---
layout: post
title:  "Scaling your dockerized webapplication"
categories: docker consul
commentIssueId: 3
---

In this article I will show you how you can build an highly scalable environment for your webapps

We are using the following tools:
* docker (get the docker toolbox: https://www.docker.com/docker-toolbox)
* consul
* haproxy
* apache with php
* Git repo with example code: https://github.com/MansM/docker-scalingdemo

You only need to install the docker toolbox, the other tools are in docker containers.

##Example usage:##
You have an webapplication that became quite popular. You want to dynamically scale up and down by adding containers.
But you dont want to update manually your haproxy configuration

##Tooling:##

###Docker:###
In the current IT environment there is never enough speed, VM's are slow and containers are hot. Docker is currently the most popular
container software.

###Haproxy:###


###Consul:###

###Apache with php:###
Just an example of an webapplication, but the you can use any other webapplication you like

##sources:##
http://sirile.github.io/2015/05/18/using-haproxy-and-consul-for-dynamic-service-discovery-on-docker.html
http://technologyconversations.com/2015/07/02/scaling-to-infinity-with-docker-swarm-docker-compose-and-consul-part-14-a-taste-of-what-is-to-come/