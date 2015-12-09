---
layout: post
title:  "Extending your HA over multiple machinesr"
date: 2015-12-05 18:00:00 +0100
categories: 
 - Docker
 - consul
 - nomad
tags:
 - Docker
 - consul
 - nomad 
commentIssueId: 1
---

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
