---
layout: post
title:  "Starting fresh with Jenkins 2.0"
date: 2016-04-27 18:00:00 +0100
categories: 
 - Docker
 - Jenkins
tags:
 - Docker
 - Jenkins
commentIssueId: 2
---
**Now Jenkins 2.0 got recently released it was time for me to start using it. Just changing the tag number of the Docker image was not enough. In this 
post I will describe what you need to do to get started.**

{::options parse_block_html="true" /}
<div class="toc">
<h4>Contents</h4>

* TOC
{:toc}

</div>

##Tooling##

- Docker
- Text editor


##Dockerfile:
As most Docker projects you need a Dockerfile. We are using the official [Jenkins](https://hub.docker.com/_/jenkins/) Dockerimage as a base.
Jenkins relies heavily on plugins. To add plugins 
{% highlight docker linenos %}
FROM jenkins:2.0
#plugins
COPY plugins.txt /usr/share/jenkins/plugins.txt
RUN /usr/local/bin/plugins.sh /usr/share/jenkins/plugins.txt

#config
COPY jobs/docker-webapp.xml /usr/share/jenkins/ref/jobs/docker-webapp/config.xml
COPY config.xml /usr/share/jenkins/ref/config.xml

USER root
#Adding Jenkins user to the correct groups
RUN usermod -a -G users jenkins
RUN usermod -a -G 100 jenkins

USER jenkins
{% endhighlight %}

##Jenkinsfile