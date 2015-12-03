---
layout: post
title:  "Scaling your webapplication with docker"
categories: docker consul
commentIssueId: 1
---

{::options parse_block_html="true" /}
<div class="toc">
Contents

* toc
{:toc}
</div>

You have an webapplication that became quite popular. You want to dynamically scale up and down by adding or removing instances of your
webapp. But you dont want to update manually your loadbalancer configuration. In this article I will show you how you can build an highly scalable 
environment for your webapps using docker containers, but you should be able to use it the same way on vm's.

##Tooling

- Docker: In the current IT environment there is never enough speed, VM's are slow and containers are hot. Docker is currently the most popular 
container software.
- Haproxy: Opensource load balancer for TCP and HTTP based services
- Consul: A tool for service discovery and configuration
- Apache with php: Just an example of an webapplication, but you can use any other webapplication type
- Supervisord: As we run next to the primary process a consul application in the container we need something to launch them

Of the list above you only need to install the docker toolbox. You can find it at the docker [website](https://www.docker.com/docker-toolbox)

##Overview

In the image below you can see how the environment will be. As you can see I use three different parts of consul. The agent, the server and template. 
The agent tells the server it is online and which services the machine/container has available. Consul template queries the server and replaces the 
configuration file of haproxy which it also notifies. On shutting down a webserver the consul agent notifies the server it is leaving and consul template 
will remove the server from the configuration file. 

![container overview](/images/2015-12-01-scaling-your-dockerized-webapplication/overview.png)

##Docker containers##
As mentioned before I am using docker containers in this setup. Three different containers can be identified:

- Consul server
- Haproxy with consul-template
- Webservice with consul agent

First of all, get the git repository with the example code:
{% highlight bash %}
git clone https://github.com/MansM/docker-scalingdemo.git
{% endhighlight %}

###Consul server
Change to the folder in your terminal. The consul server is the consul-server container made by gliderlabs, to start it:

{% highlight bash %}
docker run --rm --name consul -p 8500:8500 gliderlabs/consul-server -bootstrap
{% endhighlight %}

to view the consul server ui, you can browse to the docker_ip:8500, you can locate the ip with running the command:
{% highlight bash %}
docker-machine ip default 
{% endhighlight %}
(if it default is not the correct docker-machine, find the correct one with docker-machine ls)

You should now see something like:
![Consul ui](/images/2015-12-01-scaling-your-dockerized-webapplication/consul-ui.png)

###Web application
Now its time to add a webserver to the pool, first of all we need to build and run the docker container

{% highlight bash %}
docker build -t apachephp apachephp
docker run  --rm --name web -p 80:80 --link="consul" apachephp
{% endhighlight %}
When you refresh the consul-ui screen, you will see the webserver has been added. 
![Consul ui with webserver](/images/2015-12-01-scaling-your-dockerized-webapplication/consul-ui-web.png)

####Dockerfile
The main item of any docker container is the Dockerfile, this contains the building steps from which docker builds the image.
In this case we are using as base image the image from php including apache (line 1). On line 4 we install supervisord and two
utilities to get the rest of the Dockerfile working. The consul agent is downloaded and installed on line 9. The configuration for 
supervisord and consul is copied into the image on 12 and 13. The last line is the command that is executed on launching the container, 
here it is starting supervisord to launch the other processes.

{% highlight docker linenos %}
FROM php:5.6-apache
ENV CONSUL_AGENT_VERSION=0.5.2

RUN apt-get update && apt-get install -y supervisor wget unzip
RUN apt-get clean

RUN mkdir -p /var/log/supervisor
RUN mkdir -p /etc/consul.d 
RUN wget https://releases.hashicorp.com/consul/${CONSUL_AGENT_VERSION}/consul_${CONSUL_AGENT_VERSION}_linux_amd64.zip -O /tmp/consul-agent.zip && unzip /tmp/consul-agent.zip && mv consul /usr/local/bin

COPY src/ /var/www/html/
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf
COPY web.json /etc/consul.d/web.json

CMD ["/usr/bin/supervisord"]
{% endhighlight %}


####web.json
This file is the configuration or the consul agent. The service web is the selector which later will be used to identify the needed containers. Line 7 is quite important, it
makes the agent notify the server it is leaving when the containter is stopped. This will save you from deregistering every stopped node in the consul ui.
{% highlight json linenos %}
{
  "service": {
    "name": "web",
    "tags": ["php"],
    "port": 80
  },
  "leave_on_terminate": true
}
{% endhighlight %}

####supervisord.conf
In the configuration file for supervisord we mention that supervisord shouldnt be started as a deamon. This will make the output be shown directly
{% highlight ini linenos %}
[supervisord]
nodaemon=true

[program:apache]
command=/usr/local/bin/apache2-foreground

[program:consulagent]
command=consul agent -data-dir=/tmp/consul -join consul -config-dir /etc/consul.d
{% endhighlight %}

###Load balancer
The final container in the setup is the load balancer. 
{% highlight bash linenos %}
docker build -t loadbalancer haproxy
docker run --rm -p 80:80 -p 9000:9000 --link="consul" --link="web" --name loadbalancer loadbalancer
{% endhighlight %}
####Dockerfile
{% highlight docker linenos %}
FROM debian:latest

MAINTAINER Mans Matulewicz

ENV CONSUL_TEMPLATE_VERSION=0.10.0

RUN apt-get update && apt-get install -y haproxy wget unzip supervisor
RUN ( wget --no-check-certificate https://github.com/hashicorp/consul-template/releases/download/v${CONSUL_TEMPLATE_VERSION}/consul-template_${CONSUL_TEMPLATE_VERSION}_linux_amd64.tar.gz -O /tmp/consul_template.tar.gz && gunzip /tmp/consul_template.tar.gz && cd /tmp && tar xf /tmp/consul_template.tar && cd /tmp/consul-template* && mv consul-template /usr/bin && rm -rf /tmp/* )
RUN apt-get clean

RUN mkdir -p /var/log/supervisor

COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf
COPY haproxy.ctmpl /etc/haproxy.ctmpl
COPY haproxy.json /etc/haproxy.json

EXPOSE 80/tcp 9000/tcp
CMD ["/usr/bin/supervisord"]
{% endhighlight %}
####haproxy.cfg/ctml
{% highlight nginx linenos %}
{% raw %}
backend nodes
    mode http
    option forwardfor
    http-request set-header X-Forwarded-Port %[dst_port]
    http-request add-header X-Forwarded-Proto https if { ssl_fc }
    option httpchk HEAD / HTTP/1.1\r\nHost:localhost
    balance roundrobin {{range service "web"}}
    server {{.Node}} {{.Address}}:{{.Port}}{{end}}{% endraw %}
{% endhighlight %}
####haproxy.json
{% highlight javascript linenos %}
template {
  source = "/etc/haproxy.ctmpl"
  destination = "/etc/haproxy/haproxy.cfg"
  command = "haproxy -f /etc/haproxy/haproxy.cfg -sf $(pidof haproxy) &"
}
{% endhighlight %}
####supervisord.conf
{% highlight ini linenos %}
[supervisord]
nodaemon=true

[program:consultemplate]
command=consul-template -config=/etc/haproxy.json -consul=consul:8500
{% endhighlight %}
All the containers are listed in the docker-compose.yml. This file is used by docker-compose to run the multi-container environment.

##Docker-compose

##Sources:##

- http://sirile.github.io/2015/05/18/using-haproxy-and-consul-for-dynamic-service-discovery-on-docker.html
- http://technologyconversations.com/2015/07/02/scaling-to-infinity-with-docker-swarm-docker-compose-and-consul-part-14-a-taste-of-what-is-to-come/