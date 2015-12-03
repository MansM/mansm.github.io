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

##Tooling##

- Docker: In the current IT environment there is never enough speed, VM's are slow and containers are hot. Docker is currently the most popular container software.
- Haproxy: Opensource load balancer for TCP and HTTP based services
- Consul: A tool for service discovery and configuration
- Apache with php: Just an example of an webapplication, but the you can use any other webapplication you like
- Supervisord: As we run next to the primary process a consul application in the container we need something to launch them

Of the list above you only need to install the docker toolbox. You a can find it at the docker [website](https://www.docker.com/docker-toolbox)

Overview
In the image below you can see how the environment will be setup. As you can see I use three different parts of consul.

![container overview](/images/2015-12-01-scaling-your-dockerized-webapplication/overview.png){:width="600px"}

##Docker containers##
As mentioned before I am using docker containers in this setup. Three different containers can be identified:

- Consul server
- Haproxy with consul-template
- Webservice with consul agent

First of all, get the git repo with the example code:
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
docker run  --rm --name apachephp -p 80:80 --link="consul" apachephp
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
{% highlight ini linenos %}
[supervisord]
nodaemon=true

[program:haproxy]
command=/usr/local/bin/apache2-foreground

[program:consulagent]
command=consul agent -data-dir=/tmp/consul -join consul -config-dir /etc/consul.d
{% endhighlight %}

###Load balancer

####Dockerfile

####haproxy.cfg/ctml

####haproxy.json

####supervisord.conf

All the containers are listed in the docker-compose.yml. This file is used by docker-compose to run the multi-container environment.



##Sources:##

- http://sirile.github.io/2015/05/18/using-haproxy-and-consul-for-dynamic-service-discovery-on-docker.html
- http://technologyconversations.com/2015/07/02/scaling-to-infinity-with-docker-swarm-docker-compose-and-consul-part-14-a-taste-of-what-is-to-come/