---
layout: page
title: Operator Guide
---

##Installing XOS

##Installing OpenStack

This section describes how to bring up OpenCloud's version of an OpenStack cloud on a local cluster.  

###Cluster architecture

![Figure 1. OpenCloud cluster architecture.]({{ site.url }}/figures/controller.jpg)

Figure 1. OpenCloud cluster architecture.

The above figure shows the goal of the installation process.  At top is a controller node running 10 VMs attached to a private management network; each VM is hosting a service needed by OpenStack.  Below are compute nodes running the nova-compute and neutron-plugin-openvswitch agents.  The compute nodes connect to a publicly routable network.  

One VM on the controller node (denoted "Juju/router") serves as a router for traffic flowing between the compute nodes and the OpenStack controller services.  The Juju/router VM has network interfaces on both the public and the private management network, and is on the same IP subnet as the compute nodes.  Forwarding rules in the VMs and on the nodes enable packets to be exchanged between networks.

###Configuring the physical servers and network

The controller and compute nodes should meet the following *minimum* hardware requirements: 

* 12 CPU cores, x86_64 architecture
* 48GB RAM
* 3TB disk
* 2x 1Gbps NICs

The nodes should be installed with Ubuntu 14.04 LTS.  Both NICs should be wired to a public network; NIC1 should have a public IP address and NIC2 should be left unconfigured.  The compute nodes should not be behind a firewall.  If the controller node is behind a firewall, the following TCP ports should be opened for XOS: 22, 3128, 5000, 8080, 8777, 9292, 9696, 35357. 

###Setting up virtual infrastructure using the EC2 Install Cloud

The controller node architecture shown in Figure 1 runs each OpenStack service in its own VM, with all VMs connected by a private management (virtual) network. To easily bring up this virtual infrastructure we leverage an OpenStack cloud with its controller services running in Amazon EC2 - the "Install Cloud".  The following instructions are for OpenCloud admins that have access to the Install Cloud.  At a high level the steps are:

0. Networking setup
2. Add the server to the Install Cloud as a compute node
3. Create the management virtual network and VMs

#### Networking setup

In order to allow the new node to communicate with the Keystone and Neutron services running in EC2, it’s necessary to add a couple of security group rules for the new node.  Log into the EC2 console at Amazon AWS and add the following rules for the new node's IP address.  Look at existing rules in each security group for examples. 

* In security group *juju-amazon-6*: Add GRE tunnel rule
* In security group, *juju-amazon-3*: Add TCP/35357 rule 

On the node, add the following entries to */etc/hosts*:

```
54.165.170.148  ip-172-31-34-147.ec2.internal juju.amazon
54.165.40.107   ip-172-31-35-79.ec2.internal  keystone.amazon
54.165.71.183   ip-172-31-3-137.ec2.internal  rabbitmq.amazon
54.209.16.51    ip-172-31-4-171.ec2.internal  nova_cc.amazon
54.164.16.21    ip-172-31-26-103.ec2.internal mysql.amazon
54.88.254.182   ip-172-31-16-210.ec2.internal glance.amazon
54.209.88.253   ip-172-31-41-186.ec2.internal neutron.amazon
```

#### Add the server to the Install Cloud as a compute node 

Log into the Juju VM (*ubuntu@54.88.138.52*).  Before proceeding, make sure that you can login to the server from this VM; then run the following commands:

```
$ juju add-machine ssh:ubuntu@<server IP address>
$ juju add-unit nova-compute --to <juju id of server>
```

#### Create the management network and VMs

###Deploying the OpenStack controller services

* Install and configure Juju 
* Use Juju to install other services

###Configuring remote OpenStack clients 

Port forwarding on the Juju/router VM enables remote clients to connect to the OpenStack services on the cluster.  An OpenStack client connecting to the VM's public IP address has its request forwarded to the private IP address of the appropriate VM.  A firewall in the Juju/router VM ensures that only authorized clients are able to connect.  

When connecting to an OpenStack service, many OpenStack client libraries fetch its endpoint information from Keystone.  The OpenStack controller services register their private IP addresses on the management network with Keystone.  If a client is not connected to the management network, then it may be necessary to translate this private IP address to the public IP address used for port forwarding.  One way to do this is with iptables.  For example, if the cluster's management network is on the 192.168.100.0/24 subnet, and the public IP address for port forwarding is 1.2.3.4, then one could add the following iptables rule on the client machine:

```
$ iptables -t nat -A OUTPUT -p tcp -d 192.168.100.0/24 -j DNAT --to-destination 1.2.3.4
```

##Installing OpenVirteX

##Monitoring

##Operator View

