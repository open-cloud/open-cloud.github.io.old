---
layout: page
title: Operator Guide
---

This guide describes how to install and operate an XOS-based cloud. It
draws heavily from an existing operational cloud (OpenCloud), but with
the intent of documenting how others might replicate OpenCloud on their
own infrastructure.

##Installing XOS

See Section [Configuration Management](../2_developer/#config-mgmt) of
the Developer Guide for instructions on configuring and installing
XOS. The discussion in that section presumes each configuration is
installed on some target hardware platform that is suitable for
development (e.g., CloudLab or a CORD POD).

For operational deployments of XOS, the target hardware is likely to
be one or more backend OpenStack clusters. Information on bringing up
an OpenStack cluster is given below (Section [Installing
OpenStack](#install-openstack)). 

Information on connecting XOS to an existing OpenStack cluster is
given in Sections [Administering a
Deployment](../1_user/#admin-deployment) and [Administering a
Site](../1_user/#admin-site) of the User's Guide. These two sections
explain how to configure a Deployment to know about a set of OpenStack
clusters and how to configure a Site to know about a set of Nodes,
respectively.

##<a name="install-openstack">Installing OpenStack</a>

This section describes how to bring up XOS's version of an OpenStack
cloud on a cluster. See Sections [Administering a
Site](../1_user/#admin-site) and [Administering a
Deployment](../1_user/#admin-deployment) of the User's Guide for
instructions on importing an OpenStack cluster into XOS.

###Cluster Architecture

Figure 1 shows the goal of the installation process.  At the top is a
controller node running 10 VMs attached to a private management
network; each VM hosts a service needed by OpenStack.  Below are
compute nodes running the nova-compute and neutron-plugin-openvswitch
agents.  The compute nodes connect to a publicly routable network as
well as the management network.

![Figure 1. XOS cluster architecture.]({{ site.url }}/figures/controller.jpg)

Figure 1. XOS Cluster Architecture.

The private management network is an IP subnet with a private IP address 
space (e.g., 192.168.122.0/24).  VMs on the controller node connect to this
network via a Linux bridge.  Compute nodes on the local network can route
packets to VMs by adding a rule to their forwarding tables.  Select ports are
forwarded from the controller node's public IP address to VMs so that 
OpenStack services can be contacted by a remote client.

###Configuring Physical Servers and Network

The controller and compute nodes should meet the following *minimum*
hardware requirements:

* 12 CPU cores, x86_64 architecture
* 48GB RAM
* 3TB disk
* 2x 1Gbps NICs

The nodes should be installed with Ubuntu 14.04 LTS.  Both NICs should
be wired to a public network; NIC1 should have a public IP address and
NIC2 should be left unconfigured.  The compute nodes should not be
behind a firewall.  If the controller node is behind a firewall, the
following TCP ports should be opened for XOS: 22, 3128, 5000, 8080,
8777, 9292, 9696, 35357.

###Installing OpenStack on a Local Cluster

The controller node architecture shown in Figure 1 runs each OpenStack
service in its own VM, with all VMs connected by a private management
(virtual) network. An Ansible playbook that automates bringing up
this virtual infrastructure is available on GitHub:

https://github.com/andybavier/opencloud-cluster-setup

Consult the README.md file for instructions on how to run this playbook
and customize it for your own needs.

####Service VMs

Once you have set up the head node of your local OpenCloud cluster using 
the above scripts, you can use **virsh list** to see a list of the running 
VMs, each named after the service it hosts:

```
$ virsh list
Id    Name                    State
----------------------------------------------------
3     juju                    running
4     mysql                   running
5     rabbitmq-server         running
6     keystone                running
7     glance                  running
8     nova-cloud-controller   running
9     quantum-gateway         running
10    openstack-dashboard     running
11    ceilometer              running
12    nagios                  running
16    xos                     running
```

All of the VMs are attached to bridge `virbr0` with private addresses on the 192.168.122.0/24 subnet, and so are not reachable externally.  The IP addresses of the VMs are in `/etc/hosts`, or can be obtained using **uvt-kvm ip <VM name>**:

```
$ uvt-kvm ip xos
192.168.122.37
```

Log in to a VM using **ssh ubuntu@\<VM name\>**.  The default SSH key 
for the admin user (`/home/admin/.ssh/id_rsa.pub`) has been added for 
the ubuntu user inside all the VMs, so this should just work:

```
$ ssh ubuntu@xos
Welcome to Ubuntu 14.04.2 LTS (GNU/Linux 3.13.0-49-generic x86_64)
...
ubuntu@xos:~$
```

###Configuring Remote OpenStack Clients 

Port forwarding on the controller node enables remote clients to
connect to the OpenStack services on the cluster.  An OpenStack client
connecting to the VM's public IP address has its request forwarded to
the private IP address of the appropriate VM.  A firewall on the
controller node ensures that only authorized clients are able to
connect.

When connecting to an OpenStack service, many OpenStack client
libraries fetch its endpoint information from Keystone. The OpenStack
controller services register their private IP addresses on the
management network with Keystone.  If a client is not connected to the
management network, then it may be necessary to translate this private
IP address to the public IP address used for port forwarding.  One way
to do this is with iptables.  For example, if the cluster's management
network is on the 192.168.100.0/24 subnet, and the public IP address
for port forwarding is 1.2.3.4, then one could add the following
iptables rule on the client machine:

```
$ iptables -t nat -A OUTPUT -p tcp -d 192.168.100.0/24 -j DNAT --to-destination 1.2.3.4
```

The XOS install scripts enable SSL for the OpenStack endpoints,
using a certificate generated by Juju.  Fetch the certificate from
*/etc/ssl/certs/keystone_juju_ca_cert.pem* in the
nova-cloud-controller VM and add it to the local certificate repository
on the client (e.g., */usr/local/share/ca-certificates/*).

##Operator Tools

There is currently no comprehensive operator view. Instead, operators
use the following combintation of views and tools to a monitor and
operate an XOS deployment:

* The *Developer* view gives administrators read/write access to the
  entire data model. The *Admin-Only* tab on the *Sites*, *Slices* and
  *Users* pages gives operators access to the underlying *Controller*.

* A hidden xoslib-based alternative to the *Developer* view is
  available at *opencloud.us/admin/dashboard/xosAdminDashboard*.

* A *Nagios* view provides access to a Nagios service running on the
  head node of each underlying OpenStack cluster.

There are also a set of scripts that can be used to monitor the health
of the OpenStack services running on each cluster. On
*beta.opencloud.us*, admin credentials for all the clusters can be
found in */home/ubuntu/acb/*. The *openstack-command.sh* script can
be used to run an OpenStack command across all of the clusters and
print the output. For example, to view the status of the Nova services
and Neutron agents on all clusters:

```
$ ./openstack-command.sh "nova service-list"
$ ./openstack-command.sh "neutron agent-list"
```

To show all VMs created across all clusters:

```
$ ./openstack-command.sh "nova list --all-tenants"
```

Note that glance commands require the --os-cacert argument:

```
$ ./openstack-command.sh "glance --os-cacert /etc/ssl/certs/ca-certificates.crt image-list"
```

Andy is the maintainer of these credential files and scripts; let him
know if something is not working as expected. 

##Troubleshooting

**Symptom:** Can't create VMs on the nodes.  The *nova service-list* command shows all nova-compute instances as *down*. Additionally, XOS may display "timed out while waiting for node" in the backend_status field of the affected instances.

**Fix:** It seems that RabbitMQ is usually the culprit.  Follow these steps:

* Restart the rabbitmq-server VM.  Wait for it to come back up.

``` 
$ ssh ubuntu@rabbitmq-server "sudo shutdown -r now"
```

* Restart the nova-api-metadata service in the quantum-gateway VM.

```
$ ssh ubuntu@quantum-gateway "sudo service nova-api-metadata restart"
```

* Restart the nova-compute service on all the compute nodes. 

```
$ ssh ubuntu@<compute-node> "sudo service nova-compute restart"
```

**Symptom:** Metadata service is slow and returns *500 Internal Server Error*

**Fix:** See *Can't create VMs on the nodes.*. It's the same rabbitmq problem, and the errors are coming from the nova-api-metadata service. 

**Symptom:** When SSHing to an Instance, "This is nc from the netcat-openbsd package" is printed along with the netcat syntax.

**Fix:** You may be trying to SSH to the NAT interface of an Instance that's configured with a public IP instead of NAT.

* SSH to the Public IP Address instead. 

**Symptom:** Instance are unreachable via network, but are running on the host.

**Diagnostic Steps:** Use VNC to view console of broken instance. 

* on host physical machine, run "virsh vncdisplay <instance_name>". Note the vnc console number, add 5900 to it to get the VNC port.

* on host phyical machine, make sure port is open: "iptables -A INPUT -m state --state NEW -m tcp -p tcp --dport 5900  -j ACCEPT"

* from admin machine, setup an SSH tunnel: ssh -o "GatewayPorts yes" -L 5900:localhost:5900 ubuntu@<hostname>

* establish vnc session to localhost:5900.

**Symptom:** Interfaces become unreachable inside of instances. When inspected using VNC, instance shows UDP checksum errors
 relating to DHCP packets.

**Fix:** Older versions of dhclient are incompatible with checksum offloading on host. 

* Upgrade guest image to a newer version of dhclient
