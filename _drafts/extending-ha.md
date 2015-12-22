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

**In the last article we discussed how you can scale your webapplication with Docker. In this article I will show you how you can scale your
Docker environments over multiple servers and/or datacenters using Nomad.**

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
- Vagrant: Wrapper around virtualbox, which allows you to script vm provisioning
- Webapp & haproxy: basic 2-tier application which is described in the previous post

##Overview
To create an high available environment we need to make sure all parts are redundant available. 

NomadServer1
Nomad Server
Consul Server(container) (network overlay-consul)

DockerServer1
Nomad Client
Docker engine:
	- haproxy (network overlay-consul) (network overlay-web)
	- webapps (network overlay-consul) (network overlay-web)

NomadServer2
Nomad Server
Consul Server(container) (network overlay-consul)

DockerServer2
Nomad Client
Docker engine:
	- haproxy (network overlay-consul) (network overlay-web)
	- webapps (network overlay-consul) (network overlay-web)


TODO consul en nomad clusteren