---
layout: post
title:  "Building a homelab with Kubernetes and Raspberry pi's"
date: 2018-09-30 18:00:00 +0100
categories: 
 - Kuberetes
 - Raspberry
 - rpi
tags:
 - Kuberetes
 - Raspberry
 - rpi
commentIssueId: 4
---
**Building a homelab using modern tools/methods like Kubernetes, GitOps and Raspberry Pi's**

{::options parse_block_html="true" /}
<div class="toc">
<h4>Contents</h4>
* toc
{:toc}
</div>

# Why
In my current job I work a lot with container technologies, and for a lot of items I want to automate the stuff I use at home with the same tech. Unfortunatley, I don't have the cash for racks full of servers, SAN's, fiber switches etc etc (my girlfriend would also kill me). So I had to scale down a bit and reuse the equipment I already have. 

# What
If your home is a bit like mine, you probably run some services. Mediaplayers, mediaservers, domotica, some websites, etc. I had some spare raspberries laying around so I decided to install kubernetes on them. Kubernetes is an open-source system for deployment, scaling container workloads. Why is this important. If you run just containers on one host it's easy, but when you do it on multiple you need some system to do it for you.   

# Who
This is easy: you

# Hardware I used
- Raspberry Pi 2b
- Raspberry Pi 3b
- Simple network switch
- Synology DS1515+ (for persistent storage)
- Micro USB power cables
- Flat ethernet cables (as short as possible)

# The setup
<<TODO: create diagram>>
In this tutorial we will 2 raspberry pi's. 
# Prerequisites
First download the latest Hypriot linux build for your raspberry and write them to the sd cards. 
I recommend to give your raspberries static IP's or give them static IP's through DHCP. Log in to the raspberries and
give them a nice hostname (when you are at it, dont forget to update). Kubernetes takes the amount of available memory in account, so we need to disable swap. Do this by removing the swap entry from /etc/fstab and the command: ```swapoff -a```. You have to do this on all hosts. 

Install the following components on all:
```
apt install -y  curl software-properties-common apt-transport-https
```

Hypriot comes with docker already installed but if you used a different OS image or using vagrant you probably need to install docker yourself.
```

curl -fsSL https://download.docker.com/linux/debian/gpg | apt-key add -
apt-add-repository "deb https://download.docker.com/linux/debian $(lsb_release -cs) stable"
apt update -y && apt install -y docker-ce
```



# Installing the master
First we need to install kubeadm. Kubeadm is the tool that manages the lifecycle of a Kubernetes cluster. As kubeadm is not in the default repositories we need to install the kubernetes repository first, before we can use it we also need to add the GPG key:  



```
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
```
<br/>
When we have the repository available we can install kubeadm with the following command: ```apt install -y kubeadm```  
Kubeadm expects already to have an available kubelet. Kubelet is the component on every kubernetes machine (masters and workers) that starts the pods.
Install kubelet: ```apt install -y kubelet```. Note: when you are using Vagrant you need to edit the kubelet configuration to use the secondary nic:
open ```/etc/default/kubelet``` in your favorite editor and make sure the following line is present: ```KUBELET_EXTRA_ARGS=--node-ip=192.168.10.10```, where 192.168.10.10 is the ip of the secondairy nic. Restart kubelet afterwards (```systemctl restart kubelet```).
<<TODO: test without restarting kubelet just yet>>

The full init process takes a while and can sometimes fail on pulling images. To reduce the chance of this happening, we can prepull the images with:
```kubeadm config images pull``` Run this until all images are successfully pulled.
<<TODO: insert screenshot of kubeadm prefetch>>  

Now comes the big step, we will start the initialization process of the cluster. We do this with running the following command:
```
kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --kubernetes-version=1.12.1
```

<<TODO: --ignore-preflight-errors=all>>
when using vagrant you need to add: ```--apiserver-advertise-address=192.168.10.10```, where 192.168.10.10 is the ip of the secondairy nic.
![Kubeadm init output](/images/2018-11-homelab/kubeadm_init.png)

As the the Kubernetes apiserver starts slower then amount of it time it has to start kubelet will kill the starting container and tries again. To prevent this, we will need to update the deployment manifest of the apiserver to make sure it gets more time. Open /etc/kubernetes/manifests/kube-apiserver.yaml in your favorite editor (or just use sed) and find this line: ```failureThreshold: 8```, replace the 8 with 20 and save. Now we need to wait or kill the apiserver by ourselves. To kill the apiserver you can use the command: ```docker ps -q --filter ancestor=k8s.gcr.io/kube-apiserver:v1.12.1|xargs docker kill```, where v1.12.1 is the same as version mentioned above. Or if you want to do it manually, lookup the docker container with docker ps and find the one that has "kube-apiserver --au…" in the command colomn and kill the container with ```docker kill containerID=first_column_of_previous_command```. Now you its waiting until it finnishes, the output should look like the screenshot above.

# Installing the node (or multiple)
Installing the nodes is a lot similar to installing the masters. We start by installing kubeadm and kubelet (```apt install -y kubelet kubeadm```).  Note: when you are using Vagrant you need to edit the kubelet configuration to use the secondary nic:
open ```/etc/default/kubelet``` in your favorite editor and make sure the following line is present: ```KUBELET_EXTRA_ARGS=--node-ip=192.168.10.20```, where 192.168.10.10 is the ip of the secondary nic. Restart kubelet afterwards (systemctl restart kubelet). In the output of the init command of the master a line came how you can have the node join the master. In case you lost this output or you have to wait too long (it expires after 24 hours), you can do it as well with the following command on the master:
```
kubeadm token create  --print-join-command
```
On the node we will run the output of the command in the join command
![Kubeadm join output](/images/2018-11-homelab/kubeadm_join.png)

# Configuring kubectl
To access the newly Kubernetes cluster we will use the commandline tool called kubectl. Most likely its already installed, but if not you can install it with
```apt install -y kubectl```. Kubectl does need to know where to find the Kubernetes apiserver, we can configure this by telling kubectl where the config is located. On the master run ```export KUBECONFIG=/etc/kubernetes/admin.conf``` (it can be smart to put it in your profile). As I am a lazy person I have created an alias called k for kubectl: ```alias k=kubectl``` (you can also save this in your profile). 

# Networking
As Kubernetes allows you to choose which networking plugin you want to use, it comes default without. This is why you nodes show as NotReady:
![Kubectl get nodes (notReady)](/images/2018-11-homelab/kubectl_get_nodes_not_ready.png)


We can fix this by installing flannel (I have chosen Flannel as it comes with auto enablers for multiple platforms (amd64/arm/etc)). On master1 run:
```kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml```
After a short while your nodes will turn to ready:  
![Kubectl get nodes (notReady)](/images/2018-11-homelab/kubectl_get_nodes_ready.png)

<<TODO: ingress>>

# Finishing touches
- helm?
- 

# Last remarks
As raspberries have a filesystem on an sd flashcard, dont expect it to run forever. Dont keep data here that is really precious to you. 

# What I still want to do
- Adding monitoring
- Adding logging
- Using PoE powered Raspberry Pi 3b+
- Using MetalLB with Ubiquiti USG for loadbalancing

