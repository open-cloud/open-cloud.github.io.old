---
layout: page
title: Operator Guide
---

This guide describes how to install and operate an XOS-based cloud. It
draws heavily from an existing operational cloud (OpenCloud), but with
the intent of documenting how others might replicate OpenCloud on their
own infrastructure.

##Installing XOS

##Configuring XOS

There is an XOS Configuration File at /opt/planetstack/plstackapi_config. This
file may be edited with any text editor.

### Disabling Monitoring

To disable monitoring, add the following section to the config file:

    [gui]
    disable_minidashboard=True

##Installing OpenStack

This section describes how to bring up OpenCloud's version of an
OpenStack cloud on a cluster. See Sections [Administering a
Site](../1_user/#admin-site) and [Administering a
Deployment](../1_user/#admin-deployment) of the User's Guide for
instructions on importing an OpenStack cluster into OpenCloud.

###Cluster Architecture

Figure 1 shows the goal of the installation process.  At the top is a
controller node running 10 VMs attached to a private management
network; each VM hosts a service needed by OpenStack.  Below are
compute nodes running the nova-compute and neutron-plugin-openvswitch
agents.  The compute nodes connect to a publicly routable network.

![Figure 1. OpenCloud cluster architecture.]({{ site.url }}/figures/controller.jpg)

Figure 1. OpenCloud Cluster Architecture.

One VM on the controller node (denoted "Juju/router") serves as a
router for traffic flowing between the compute nodes and the OpenStack
controller services.  The Juju/router VM has network interfaces on
both the public and the private management network, and is on the same
IP subnet as the compute nodes.  Forwarding rules in the VMs and on
the nodes enable packets to be exchanged between networks.

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

###Using the Install Cloud

The controller node architecture shown in Figure 1 runs each OpenStack
service in its own VM, with all VMs connected by a private management
(virtual) network. To easily bring up this virtual infrastructure we
leverage another OpenStack cloud that runs its controller services in
Amazon EC2. We call the *Install Cloud*, and we use it to bootstrap
individual OpenStack clusters, where the compute nodes in the Install
Cloud are also the head nodes in individual OpenStack clusters.

The following instructions are for OpenCloud admins that have access
to the Install Cloud.

####Networking Setup

To allow the new node to communicate with the Keystone and Neutron
services running in EC2, it is necessary to add a couple security
group rules for the new node.  Log into the EC2 console at Amazon AWS
and add the following rules for the new node's IP address.  Look at
existing rules in each security group for examples.

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

####Add the Server to the Install Cloud as a Compute Node 

Log into the Juju VM (*ubuntu@54.88.138.52*).  Before proceeding, make
sure that you can login to the server from this VM; then run the
following commands:

```
$ juju add-machine ssh:ubuntu@<server IP address>
$ juju add-unit nova-compute --to <juju id of server>
```

After the nova-compute service is running on the node, it is necessary
to install some OpenCloud-specific configuration.  Add information for
the new head node to *~/opencloud-install/ansible/amazon.hosts* and
run:

```
$ cd ~/opencloud-install/ansible
$ ansible-playbook -i inventory/amazon.hosts compute.yml
```

This will likely cause the node to reboot. 

####Create the Management Network and VMs

Log into the Juju VM (*ubuntu@54.88.138.52*).  To configure VMs for
hosting the new cluster's OpenStack controller services, run the
following: 

```
$ cd ~/opencloud-install/ansible
$ ansible-playbook -i inventory/amazon.hosts site-setup.yml
```

After the VMs have been created on the new head node, add information
for the Juju VM to *~/opencloud-install/ansible/inventory/juju.hosts*
and *~/.ssh/config*.  Then run:

```
$ cd ~/opencloud-install/ansible
$ ansible-playbook -i inventory/juju.hosts juju-preinstall.yml
```

(This step requires a bit of cleanup to be actually as simple as
described above.)

###Deploying OpenStack Controller Services

At this point, the VMs and VNs required by the new site should be in
place but they are not running any services.  Next we need to install
the OpenStack controller services in the VMs.

Log into the Juju VM on the new head node.  Create a basic
*/etc/ansible/hosts* file containing a *[service]*  group.  Add the
private IP addresses of all VMs to this group.  An example:

```
[service]
192.168.6.2
192.168.6.[4:11]
```

Next, create a "XXXX-preinstall.yml" playbook, where XXXX is replaced
by the name of the cluster.  Copy one of the existing preinstall
playbooks and change the variables as appropriate for the new cluster.
To install and configure Juju in the VM, run that playbook:

```
$ ansible-playbook foobar-preinstall.yml
$ juju generate-config
```

Now Juju can be used to deploy the controller
services.  First, edit *~/.juju/environments.yaml* using the following
as a template:

```
default: mpisws

environments:
    mpisws:
        type: manual
        bootstrap-host: juju
        bootstrap-user: ubuntu
        default-series: trusty
        apt-http-proxy: http://192.168.6.2:8000
```

Next, add the local VMs to the local Juju environment:

```
$ juju add-machine ssh:192.168.x.y
```

Once all VMs are added to Juju, edit
*~/opencloud-install/juju/manual-install-controller-services.py* so
that it contains the correct Juju IDs and then run it:

```
$ cd ~/opencloud-install/juju
$ ./manual-install-controller-services.py
```

Wait for all services to be installed and started, then run:

```
$ ./manual-install-controller-relations.py
```

To perform the final configuration steps for the site, create
*~/opencloud-install/ansible/XXXX.yml*, replacing XXXX with the name
of your cluster, using an existing file as
a template.  Run that playbook using *juju-ansible-playbook*:

```
$ cd ~/opencloud-install/ansible
$ juju-ansible-playbook foobar.yml
```

Now the compute nodes can be added to the cluster.

###Deploying OpenStack Compute Nodes

This step assumes a pool of servers in the cluster with Ubuntu 14.04
LTS installed, that are accessible via SSH from the Juju VM.  It is
necessary to do some initial configuration before adding them to
Juju.  Edit */etc/ansible/hosts* so that it contains a *[compute]*
group containing the server names, for example:

```
[compute]
node[62:68].mpisws.vicci.org
```

Next run the preinstall playbook again:

```
$ ansible-playbook foobar-preinstall.yml
```

Now the servers can be added to Juju using *juju add-machine*.  Once
they are all added, edit
*~/opencloud-install/juju/manual-install-nova-compute.py* so that it
contains the Juju ID of one of the servers and run it:

```
$ cd ~/opencloud-install/juju
$ ./manual-install-nova-compute.py
```

Once the nova-compute service is running on that node, it can be added
to the other nodes by their Juju IDs as follows:

```
$ juju add-unit nova-compute --to <Juju ID>
```

Finally, re-run the main cluster playbook created earlier using
*juju-ansible-playbook*: 

```
$ juju-ansible-playbook foobar.yml
```

At this point you can start testing the new OpenStack cluster to make
sure that it's working.

###Configuring Remote OpenStack Clients 

Port forwarding on the Juju/router VM enables remote clients to
connect to the OpenStack services on the cluster.  An OpenStack client
connecting to the VM's public IP address has its request forwarded to
the private IP address of the appropriate VM.  A firewall in the
Juju/router VM ensures that only authorized clients are able to
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

The OpenCloud install scripts enable SSL for the OpenStack endpoints,
using a certificate generated by Juju.  Get the certificate from
*/etc/ssl/certs/keystone_juju_ca_cert.pem* in the
nova-cloud-controller VM and append it to
*/etc/ssl/certs/ca-certificates.crt* on the client.

##Installing OpenVirteX

This section describes how to install the OpenVirteX (OVX) network
hypervisor and connect it to OpenStack's Neutron. For a more in-depth
discussion, see the OVX website: [OVX Installation
Instructions](http://ovx.onlab.us/getting-started/installation/) and
[OpenStack Neutron Installation
Instructions](http://ovx.onlab.us/openstack/).

![Figure 2. OpenCloud Neutron Architecture.]({{ site.url }}/figures/neutron.jpg)

Figure 2. OpenCloud Neutron Architecture.

An overview of the OpenCloud networking stack is shown in Figure 2. 
Each compute node has four software switches:

1. br-int (data bridge) connects the private network of the
compute instances VMs. This bridge typically has the fastest physical
interface available on the compute node.

2. br-ctl (control bridge) connects the SDN controllers to
OVX. Tenants are not allowed to connect directly to this bridge.

3. br-nat (NAT bridge) gives VMs access to the public Internet. The
admin is responsible for correct configuration beyond this bridge.

4. br-ex (external bridge) allows VMs to be reachable from the public
Internet. OpenCloud allocates an IP address from the pool of available
public IPs to each VM that requests one. (See the description of
OpenStack Neutron below for information about configuring this
pool). The admin is responsible for correct configuration beyond this
bridge.

Whenever a compute instance is spawned, it automatically connects to
the data bridge, and depending on the request, may also connect to
either the NAT or the external bridge.

Note that only the data bridges and the physical switch are in
OpenFlow mode. The remaining software switches are in L2 learning
mode.

### Cleanup Configuration State

Begin by cleaning up some of the configuration made by Juju. The
following script removes and disables unneeded OpenvSwitch components
on the compute nodes. *[Eventually, OVX will be configured through
Juju.]*

```
#!/bin/sh

# Cleanup after regular OpenCloud install
ansible -i opencloud-install/ansible/hosts onlab-oc-compute -m command -u ubuntu -s -a "/usr/bin/neutron-ovs-cleanup --ovs_all_ports"
ansible -i opencloud-install/ansible/hosts onlab-oc-compute -m command -u ubuntu -s -a "/usr/sbin/service neutron-plugin-openvswitch-agent stop"
ansible -i opencloud-install/ansible/hosts onlab-oc-compute -m command -u ubuntu -s -a "/usr/bin/ovs-vsctl del-br br-tun"
ansible -i opencloud-install/ansible/hosts onlab-oc-compute -m command -u ubuntu -s -a "/sbin/ip link delete phy-br-nat"
ansible -i opencloud-install/ansible/hosts onlab-oc-compute -m command -u ubuntu -s -a "/sbin/ip link delete int-br-nat"
ansible -i opencloud-install/ansible/hosts onlab-oc-compute -m command -u ubuntu -s -a "/sbin/ip link delete phy-br-ex"
ansible -i opencloud-install/ansible/hosts onlab-oc-compute -m command -u ubuntu -s -a "/sbin/ip link delete int-br-ex"
# Remove default libvirt network (virbr0, 192.168.122.0/24)
ansible -i opencloud-install/ansible/hosts onlab-oc-compute -m command -u ubuntu -s -a "/usr/bin/virsh net-destroy default"
```

### Configure Switches

All internal switches need to be configured in OpenFlow mode and point
to OpenVirteX. For the *br-int* switches on the compute nodes, do this
by running the following Ansible command (replace HOST and PORT by the
correct values):

```
ansible -i ansible/hosts onlab-oc-compute -m command -ubuntu -s -a "/usr/bin/ovs-vsctl set-controller br-int tcp:HOST:PORT"
```

Refer to the administrator manual on how to configure the physical
switches.

### Install OVX Software

OVX is a fairly heavy piece of software in its current implementation
and these system requirements should not be taken lightly; the number
of slices, size of the flowspace, and number of datapaths present in
your network all contribute to the scaling factors. Only the most
trivial of environments can tolerate the minimum system requirements. 
We recommended a system that has at least 4 cores and has 4 GB of
memory available for the Java heap.

Note that OVX can be deployed on any machine where all switches (both
hardware and software) can reach the machine.

Start by installing some common software packages:

```
$ sudo apt-get install git maven mongodb
```

Proceed by installing Java Oracle 7. Please do not use OpenJDK because
it will not work! Also, Java Oracly 8 is currently untested.

```
$ sudo apt-get install software-properties-common -y
$ sudo add-apt-repository ppa:webupd8team/java -y
$ sudo apt-get update
$ sudo apt-get install oracle-java7-installer oracle-java7-set-default -y
```

Now download, compile, and run OVX:

```
$ git clone https://github.com/OPENNETWORKINGLAB/OpenVirteX.git -b 0.0-MAINT
$ sh OpenVirteX/scripts/ovx.sh
```

Refer to the [OVX installation
page](http://ovx.onlab.us/getting-started/installation/) for the
available command line options.

### OpenStack Neutron

On the machine that will run the OpenStack Neutron plugin, download
the source.

```
$ git clone -b ovx https://github.com/OPENNETWORKINGLAB/neutron.git -b stable/icehouse
```

TODO: ensure source becomes default Neutron repo

Next configure OpenStack Neutron to use the OpenCloud plugin by
editing */etc/neutron/neutron.conf*:

```
core_plugin = neutron.plugins.opencloud.opencloud_neutron_plugin.OpenCloudPluginV2
```

You must also configure the plugin; at the very least, verify the
settings for the *api_host*, *of_host*, and the Nova credentials
*username* and *password*. You can do this in
*/etc/neutron/plugins/opencloud/opencloud_neutron_plugin.ini*.

```
[ovx]
username = admin                            # OVX admin user
password =                                  # OVX admin password
api_host = localhost                        # OVX RPC API server address
api_port = 8080                             # OVX RPC API server port
of_host = localhost                         # OVX OpenFlow server address
of_port = 6633                              # OVX OpenFlow server port

[ovs]
data_bridge = br-int                        # Data network bridge
ctrl_bridge = br-ctl                        # Control network bridge
nat_bridge = br-nat                         # NAT network bridge
ex_bridge = br-ex                           # External network bridge

[nova]
username = admin                            # Nova username
password =                                  # Nova password
project_id = admin                          # Nova project ID (name, not the tenant UUID)
auth_url = http://localhost:5000/v2.0/      # Nova authentication URL
image_name = ovx-floodlight                 # SDN controller image name
image_port = 6633                           # OpenFlow port of SDN controller image
flavor = m1.small                           # Machine flavor on which to run SDN controller
key_name =                                  # Name of keypair to inject into controller instance
timeout = 30                                # Number of seconds to try start the controller instance
```

Note that the *image_name* parameter in the above configuration file
refers to the default SDN controller image that will be spawned for
each virtual network that OVX creates. We have included a script in
*neutron/neutron/plugins/ovx/build-floodlight-vm.sh* to build an image
that uses the FloodLight OpenFlow controller. Run the script to build
the controller image, then import it in OpenStack Glance (note that
the given name should correspond to your plugin configuration).

```
$ glance image-create --name ovx-floodlight --disk-format=qcow2 --container-format=bare --file ubuntu-14.04-server-cloudimg-amd64-disk1.img
```

TODO: The final step is to configure a pool of public IP addresses.
Do so by opening
*neutron/neutron/plugins/opencloud/common/constants.py* and editing
the *EXT_SUBNET* section. Make sure to update the fields for *cidr*,
*gateway_ip*, and *allocation_pools*. If needed, have multiple *start*
and *end* entries if you have a fragmented IP address space. For
example:

```
'allocation_pools': [{"start": "10.0.2.3", "end": "10.0.2.15"}, {"start": "10.0.2.17", "end": "10.0.2.17"}, {"start": "10.0.2.19", "end": "10.0.2.254"}],
```

You are now ready to restart the Neutron plugin:

```
sudo service neutron-server restart
```

##Operator Tools

There is currently no comprehensive operator view. Instead, operators
use the following combintation of views and tools to a monitor and
operate OpenCloud:

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

**Symptom:** Can't create VMs on the nodes.  The *nova service-list* command shows all nova-compute instances as *down*.

**Fix:** It seems that RabbitMQ is usually the culprit.  Follow these steps:

* Restart the rabbitmq-server VM.  Wait for it to come back up.
* Restart the nova-api-metadata service in the quantum-gateway VM.
* Restart the nova-compute service on all the compute nodes.
