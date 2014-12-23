---
layout: page
title: Operator Guide
---

##Installing XOS

##Installing OpenStack

This section describes how to bring up OpenCloud's version of an OpenStack cloud on a local cluster.  These instructions assume that Ubuntu 14.04 LTS has already been installed on the local servers.  Each server should have at least 12 CPU cores and 48GB RAM.  

###Cluster architecture

![Figure 1. OpenCloud cluster architecture.]({{ site.url }}/figures/controller.jpg)

Figure 1. OpenCloud cluster architecture.

The above figure shows the goal of the installation process.  At top is a controller node running 10 VMs attached to a private management network; each VM is hosting a service needed by OpenStack.  Below are compute nodes running the nova-compute and neutron-plugin-openvswitch agents.  The compute nodes connect to a publicly routable network.  

One VM on the controller node (denoted "Juju/router") serves as a router for traffic flowing between the compute nodes and the OpenStack controller services.  The Juju/router VM has network interfaces on both the public and the private management network, and is on the same IP subnet as the compute nodes.  Forwarding rules in the VMs and on the nodes enable packets to be exchanged between networks.

Port forwarding on the Juju/router VM enables remote clients to connect to the OpenStack services on the cluster.  An OpenStack client connects to a URL containing this VM's public IP address and the request is forwarded to the private IP address of the appropriate VM.  A firewall in the Juju/router VM ensures that only authorized clients are able to connect.  

###Setting up virtual infrastructure using EC2 Install Cloud

###Deploying OpenStack controller services

##Installing OpenVirteX

##Monitoring

##Operator View

