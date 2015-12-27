---
layout: post
title:  "Extending your HA over multiple machines"
date: 2015-12-31 18:00:00 +0100
categories: 
 - Docker
 - consul
 - nomad
 - ansible
tags:
 - Docker
 - consul
 - nomad 
 - ansible
commentIssueId: 2
---
{% highlight yaml linenos %}
{% endhighlight %}
**In the last article we discussed how you can scale your webapplication with Docker. In this article I will show you how you can scale your
Docker environments over multiple servers and/or datacenters using Nomad.This post will be a focussed on building infrastructure.**

{::options parse_block_html="true" /}
<div class="toc">
<h4>Contents</h4>
* toc
{:toc}
</div>


##Tooling
- Docker: Container management software
- Nomad: A distributed, high available. datacenter-aware scheduler
- Consul: A tool for service discovery and configuration
- Ansible (version 2): platform for deploying software and configuration management
- Git: Version control system for source code
- Virtualbox: Free hypervisor
- Vagrant: Wrapper around virtualbox, which allows you to script vm provisioning. 
- Webapp & haproxy: basic 2-tier application which is described in the previous post

On your local machine you need to install vagrant, virtualbox and ansible. I can highly recommed using chocolatey on windows or brew on osx for
installing the applications. Keep in mind you need to install version 2 of ansible (brew install --devel ansible), this because I need to use some 
functionality that is not available in version 1.x . 

##Overview
To create an high available environment we need to make sure all parts are redundant available. This result in every server
being available more then once. More servers will result in higher memmory usage. I have ran this setup with ease on my 8GB 2012 MacBook Air.
In the image below the architecture of the solution is shown:

![Servers overview](/images/2016-01-scaling-your-docker-environment/overview.png)

##Ansible
I normally use puppet for my configuration management, but during this project I decided to use ansible. For a small environment ansible is very
easy to use and gets the job done. As this is my first ever usage of ansible, I will probably have created some code that should be done 
differently. Feel free to give feedback.

##Nomad
As mentioned earlier, nomad is a scheduler. Nomad was designed with Docker in mind, but also allows you to run "system" jobs, by example docker-engine.

##Networking
In most large enterprises a lot of applications are getting webbased. This results in the dockerized webapps be available on an routeable address.
I used 10.50.x.x/16 so I have enough address space. 

##Servers

###Nomadservers
On the machines with the name nomadserver1 and nomadserver2 the following components will be installed and configured:

- consul
- bind
- nomad

All of this is done by ansible. In the playbook for this project, which is partly shown below, you can see the applications being
setup by using roles. 
{% highlight yaml linenos %}
- hosts: nomadserver*
  become: yes
  vars:
   - consul_type: "server"
  roles:
    - consul
    - bind
    - nomad

- hosts: nomadserver2
  become: yes
  tasks:
    - name: consul cluster maken
      command: /usr/bin/consul join {{ nomadserver1 }}
    - name: nomad cluster maken
      environment:
        NOMAD_ADDR: "http://{{ ansible_eth2['ipv4'].address }}:4646"
      shell: /usr/bin/nomad server-join {{ nomadserver1 }}
{% endhighlight %}

On the nomadservers the consul is running in server mode and form together a cluser. All the consul agents running in next to 
"businessapplications" are running in client mode.       


###Dockerservers
