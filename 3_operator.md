---
layout: page
title: Operator Guide
---

This guide describes how to install and operate an XOS-based cloud. It
draws heavily from an existing operational cloud (OpenCloud), but with
the intent of documenting how others might replicate OpenCloud on their
own infrastructure.

##Installing XOS

See Section [Development Environment](../2_developer/#devel) of
the Developer Guide for instructions on installing XOS. Once XOS is
installed, the next step for most installations is to connect XOS to
one or more backend OpenStack clusters. 

Information on bringing up an OpenStack cluster is given below
(Section [Installing OpenStack](#install-openstack)).

Information on connecting XOS to an existing OpenStack cluster is
given in Sections [Administering a
Deployment](../1_user/#admin-deployment) and [Administering a
Site](../1_user/#admin-site) of the User's Guide. These two sections
explain how to configure a Deployment to know about a set of OpenStack
clusters and how to configure a Site to know about a set of Nodes,
respectively.

##Configuring XOS

There is an XOS Configuration File at /opt/xos/xos_config. This
file may be edited with any text editor.

### Disabling Monitoring

Monitoring services using ceilometer are enabled by default. To disable
monitoring and the associated mini-dashboard statistics that are displayed
throughout the XOS website, add the following section to the config file:

    [gui]
    disable_minidashboard=True

##Using XOS with nginx via WSGI

The development server is sufficient for most people wishing to develop XOS,
but for production environments, we recommend running XOS behind a front-end
such as nginx.

A sample configuration file for nginx is located in the nginx subdirectory of
the XOS git repository. This config fie is setup to look for static files in
/var/www/xos/static, and that subdirectory must be created. All static files
located in the following subdirectories must be copied to /var/www/xos/static/:

    /opt/xos/core/static
    /opt/xos/core/xoslib/static
    # note that the following two paths may vary depending on Linux distribution
    /usr/local/lib/python2.7/dist-packages/Django-1.7-py2.7.egg/django/contrib/admin/static
    /usr/lib/python2.7/site-packages/suit/static

The following commands can be used to start, stop, and restart the uwsgi server:

    start: cd /opt/xos; uwsgi --start-unsubscribed /opt/xos/uwsgi/xos.ini

    stop: uwsgi --stop /var/run/uwsgi/uwsgi.pid

    restart: uwsgi --reload /var/run/uwsgi/uwsgi.pid

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
agents.  The compute nodes connect to a publicly routable network.

![Figure 1. XOS cluster architecture.]({{ site.url }}/figures/controller.jpg)

Figure 1. XOS Cluster Architecture.

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

The XOS install scripts enable SSL for the OpenStack endpoints,
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

![Figure 2. XOS Neutron Architecture.]({{ site.url }}/figures/neutron.jpg)

Figure 2. XOS Neutron Architecture.

An overview of the XOS networking stack is shown in Figure 2. 
Each compute node has four software switches:

1. br-int (data bridge) connects the private network of the
compute instances VMs. This bridge typically has the fastest physical
interface available on the compute node.

2. br-ctl (control bridge) connects the SDN controllers to
OVX. Tenants are not allowed to connect directly to this bridge.

3. br-nat (NAT bridge) gives VMs access to the public Internet. The
admin is responsible for correct configuration beyond this bridge.

4. br-ex (external bridge) allows VMs to be reachable from the public
Internet. XOS allocates an IP address from the pool of available
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

# Cleanup after regular XOS install
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

Next configure OpenStack Neutron to use the XOS plugin by
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
