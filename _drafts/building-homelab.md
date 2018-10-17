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

## Why
In my current job I work a lot with container technologies, and for a lot of items I want to automate the stuff I use at home with the same tech. Unfortonatly I dont have the cash for racks full of servers, SAN's, fiber switches etc etc (my girlfriend would also kill me). So I had to scale down a bit and reuse the equipment I already have. 

# What
If your home is a bit like mine, you probably run some services. Mediaplayers, mediaservers, domotica, some websites, etc. I had some spare raspberries laying around so I decided to install kubernetes on them. Kubernetes is an open-source system for deployment, scaling container workloads. Why is this important. If you run just containers one host its easy, but when you do it on multiple you need some system to do it for you.   

# Who
This is easy: you

# Hardware I used
- Raspberry Pi 2b
- Raspberry Pi 3b
- Simple network switch
- Synology DS1515+ (for persistent storage)
- Micro USB power cables
- Flat ethernet cables (as short as possible)

# Prerequisites
First download the latest Hypriot linux build for your raspberry and write them to the sd cards. 
I recommend to give your raspberries static IP's or give them static IP's through DHCP. Log in to the raspberries and
give them a nice hostname (when you are at it, dont forget to update). Kubernetes takes the amount of available memory in account, so we need to disable swap. Do this by removing the swap entry from /etc/fstab and the command: ```swapoff -a```. You have to do this on all hosts

# Installing the master
First we need to install kubeadm. Kubeadm is the tool that manages the lifecycle of a Kubernetes cluster. As kubeadm is not in the default repositories we need to install the kubernetes repository first, before we can use it we also need to add the GPG key:  
```
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
add-apt-repository "deb https://download.docker.com/linux/debian $(lsb_release -cs) stable"
```
<br/>
When we have the repository available we can install kubeadm with the following command: ```apt install -y kubeadm```  
Kubeadm expects already to have an available kubelet, kubelet is the component on every kubernetes machine (masters and workers) that starts the pods.
Install kubelet: ```apt install -y kubelet```. Note: when you are using Vagrant you need to edit the kubelet configuration to use the secondairy nic:
open ```/etc/default/kubelet``` in your favorite editor and make sure the following line is present: ```KUBELET_EXTRA_ARGS=--node-ip=192.168.10.10```, where 192.168.10.10 is the ip of the secondairy nic. Restart kubelet afterwards (systemctl restart kubelet).

The full init process takes a while and can sometimes fail on pulling images, to reduce the chance of this happening we can prepull the images with:
```kubeadm config images pull``` Run this until all images are successfully pulled.
<<TODO: insert screenshot of kubeadm prefetch>>  

Now comes the big step, we will start the initialization process of the cluster. We do this with running the following command:
```
kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --kubernetes-version=1.12.0
```
when using vagrant you need to add: ```--apiserver-advertise-address=192.168.10.10```, where 192.168.10.10 is the ip of the secondairy nic.
![Kubeadm init output](/images/2018-11-homelab/kubeadm_init.png)

As the the Kubernetes apiserver starts slower then amount of it time it has to start kubelet will kill the starting container and tries again. To prevent this we will need to update the deployment manifest of the apiserver to make sure it gets more time. Open /etc/kubernetes/manifests/kube-apiserver.yaml in your favorite editor (or just use sed) and find this line: ```failureThreshold: 8```, replace the 8 with 20 and save. Now we need to wait or kill the apiserver by ourselves.  
 <<TODO: how to kill apiserver>>

# Installing the node (or multiple)
Installing the nodes a lot simmilar to installing the masters. We start by installing kubeadm and kubelet (```apt install -y kubelet kubeadm```).  Note: when you are using Vagrant you need to edit the kubelet configuration to use the secondairy nic:
open ```/etc/default/kubelet``` in your favorite editor and make sure the following line is present: ```KUBELET_EXTRA_ARGS=--node-ip=192.168.10.10```, where 192.168.10.10 is the ip of the secondairy nic. Restart kubelet afterwards (systemctl restart kubelet). In the output of the init command of the master a line came how you can have the node join the master. But in case you lost this output or you have wait too long (it expires after 24 hours), you can do it as well with the following steps on the master:
```
kubeadm token create
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
```
<<TODO: insert screenshot of tokenstuff>>  

On the node we will use the output of the commands in the join command:
```
kubeadm join 192.168.10.10:6443 \
  --token <<output of the first command>> \
  --discovery-token-ca-cert-hash sha256:<<output of the second command>>
```
![Kubeadm join output](/images/2018-11-homelab/kubeadm_join.png)

# Configuring kubectl
To access the newly Kubernetes cluster we will use the commandline tool called kubectl, people like me dont like to type it fully everytime, so I tend to alias it
<<TODO: how to install kubectl>>
export KUBECONFIG=/etc/kubernetes/admin.conf

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

# What I still want to do
Adding monitoring
Adding logging
Using PoE powered Raspberry Pi 3b+
Using MetalLB with Ubiquiti USG for 

