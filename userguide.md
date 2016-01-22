---
layout: page
title: User Guide
---

{% include toc.html %}

This guide describes how users log into XOS, acquire VMs, and access
XOS services. It assumes the reader is familiar with the general
concepts presented in the [Architecture Guide](/archguide/). 

Because this guide is focused on users, we use OpenCloud as an example
of an operational cloud that users might access. This implies a
particular portal, a particular set of nodes and sites, and a
particular collection of services. Substitute local specifics for an
alternative XOS installation.

## Getting Started

Users access OpenCloud by logging into the portal at
[portal.opencloud.us](http://portal.opencloud.us). New users register
through the same portal, with the request approved by the PI
(Principle Investigator) at their home site.

Once the user has an account, the following steps are a quick guide
to utilizing OpenCloud resources:

1. **Upload a public key.** Select the *Users* tab in the left-hand
   navigation bar, click on your email address, and then paste your
   public key in the box. Click the *Save* button at right.

2. **Ask your Site PI to create a slice.** Instructions for PIs (see
   [Administering a Site](#admin-site) for more information):
   - Select the *Slices* tab in the left-hand navigation bar, and then
     the *Add Slice* button on the right. In the *Slice Details* form,
     choose a name for the slice and select your own site from the
     drop-down menu. Slice names must be unique. Click the *Save and
     continue editing* button on the right before proceeding.  
   - Select the *Privileges* tab from the top, and then click the *Add
     another slice privilege* link.  Add the users that will
     administer the slice with the Admin privilege. (These users will
     be able to extend slice privileges to other users.)  
   - Click the *Save* button when done.

3. **Create instances.** Use either the [Tenant](#tenant-view) or the
   [Developer](#developer-view) View (see next section) to instantiate
   instances (VMs) for your slice.

4. **Log into instances.** Once an instance comes up, you will be able to
   learn its *instance ID*. You can then ssh into the instance using
   this ID and the physical node it is running on. See
   [Accessing a Instance](#access-instance) for more information.

## User Views

Users interact with OpenCloud through a configurable set of *Views*,
each tailored for a different usage scenario or workflow. By default,
a user's home dashboard includes the *Tenant* view; a *Customize* tab
allows the user to add and remove views from their home dashboard,
including the more "advanced" *Developer* view. Both the Tenant and
Developer views are used to administer a user's slices. From any page,
selecting the *Home* tab in the left-hand navigation bar brings the
user back to his or her home dashboard.

Users are also able to directly navigate the underlying data model
using the left-hand navigation bar. Each tabs (Deployments,
Sites, Slices, Users, and Services) correspond to the core XOS objects, as
described in the [Data Model](/archguide/#data-model). The
following sections of this guide describe how to use these
tabs to manage users, sites, and deployments, respectively. (The views
described in this section are typically use to manage slices.)

There are also views designed to support operators, as described
in the [Operator Guide](/opsguide/).

### Tenant View

The Tenant view provides a simple means to acquire Instances, with
minimal control over the low-level details of where those Instances are
placed and what networks interconnect them. It is loosely patterned
after the Amazon EC2 interface.

The Tenant view allows users to specify the number of instances that are
to be instantiated on the available sites, select an *Image* and
*Flavor* for each instance, mount select *Data Set(s)* into each instance,
and specify the *TCP Ports* the slice is going to use. Users can also
set the *ServiceClass* for the slice, although *Best Effort* is
currently the only supported class.

In the Tenant View, click on the *SSH Commands* button to see SSH
commands that can be cut-and-pasted into the terminal for logging into
your instances; click the *Download* button to save them as a local
text file.  [Accessing an Instance](#access-instance) explains other 
ways to configure SSH access for a slice's instances.

Note that slices intially created through the Tenant view may also be
managed through the Developer view.

### Developer View

The Developer view gives users full control over how their slices are
instantiated, including instance placement, network configuration, and
the privileges granted to other users:

* **Instances:** Select the *Instances* tab at the top to manage the
  instances bound to the slice. From the resulting page, select a target
  *Deployment*, and then pick an individual *Node* at that deployment. 
  Users are also able to select an *Image* and a *Flavor* on either a
  per-instance or a per-slice basis. All deployments that the user is
  permitted to access are visible when instantiating instances.

* **Networks:** Select the *Networks* tab at the top to manage the
  networks connected to the slice. Each slice is automatically
  configured with two virtual networks (one public and one private),
  but the user has the option of connecting the slice's instances to
  additional virtual networks. (Creating additional networks and
  connecting a slice to them is currently an undocumented feature.)

* **Privileges:** Select the *Privileges* tab at the top to manage the
  users that have access to the slice. Granting a user *Admin*
  privilege means giving them the authority to modify slice
  parameters. Granting a user *Default* privilege means the giving
  them access to the slice itself, that is, the right to ssh into the
  slice's instances.

### xsh View

The xsh view provides an interactive shell through which users can
access XOS objects. It is a Javascript-based environment that includes
*xoslib*, a library projection of the XOS data model. A builtin
tutorial illustrates how to use xsh.

## Accessing an Instance

Instances connect to the network via NAT; logging into the instance relies
on SSH proxying to forward incoming SSH connections. In the Tenant View, 
click on the *SSH Commands* button to see SSH
commands that can be cut-and-pasted into the terminal for logging into
your instances. In the Developer
View, the instance Id and node name are displayed in the Instance frame.
You can use this information to add lines to your ~/.ssh/config file 
similar to the following:

```
Host foobar
  User ubuntu
  IdentityFile ~/.ssh/id_rsa
  ProxyCommand ssh -q instance-0000006c@node5.cs.arizona.edu
```

In the above, replace "foobar" with a label of your choice for this
instance.  *User* is the default login user for the image.
*IdentitiyFile* should point to the key that you've uploaded to
OpenCloud.  *ProxyCommand* should point to the instance ID and node
for the instance. Once an entry is present for the instance in
~/.ssh/config, you can login using the label:

```
# ssh foobar
```

Other utilities like scp also work as expected when referencing
the instance using the label.

A current limitation is that only one user key is injected into the
slice. Because SSH is indirect through the *ProxyCommand*, it is not
sufficient to manually add additional keys for other users to an 
account inside the instance; for the time being, an administrator will 
need to add the additional keys to the proxy environment as well.

In addition to SSH connectivity via NAT, there are two other
network-related issues of note. First, to run an Internet-accessible
service in a slice, it is necessary to reserve a TCP or UDP port.
This is done using the *Network Ports* field in the Tenant View. The
service can then be accessed at this port on the hosting server (e.g.,
*node5.cs.arizona.edu* in the above example). Second, all the instances
at a given site are automatically connected by a private network. Run
*ifconfig* from within an instance to learn the instance's private address
(i.e., the *10.x.x.x* address associated with *eth0*). This private
virtual network is per-site. Instances in different sites must use an
Internet-accessible address to communicate (i.e., using a reserved
port and hosting server name as described above).

## Administering a User

All users are able to manage their own accounts and Site Admins are
able to manage the accounts of users homed at that site. Select the
*Users* tab in the left-hand navigation bar, and then click on the 
desired user. The available user details are as follows:

* **Login Details:** Select the *Login Details* tab to set parameters
  of the user's account. These include the user's *Email Address*
  (which uniquely identifies the user), home *Site*, *Password*, and
  *Public Key* (which is used to ssh into any instances created on the
  user's behalf). Click the The *Is Active* button to enable the
  account.

* **Contact Information:** Select the *Contact Information* tab to set
    the user's name and phone number.

* **Site Privileges:**: Select the *Site Privileges* tab to set the
  site-related privileges for the user.

* **Slice Privileges:**: Select the *Slice Privileges* tab to set the
  slice-related privileges for the user.

## Administering a Site

Site Admins are responsible for managing the users, slices, nodes, and
deployments affiliated with the site. Select the *Sites* tab in the
left-hand navigation bar and click on the desired site. The available
site details are as follows:

* **Users:** Select the *User* tab to proactively add users to a site,
    as well as change information for existing users.  Alternatively,
    a Site Admin is informed via email when a user requests an
    account, and has the opportunity to either accept or deny the
    request.

* **Privileges:** Select the *Privileges* tab to grant users at the
  site Admin, PI, or Tech privileges. All users homed at the site have
  default privilege.

* **Slices:** Select the *Slices* tab to create and manage the site's
  slices. To create a new slice, select *Add Slice*, fill in the
  *Slice Name*, and select *Save and Continue Editing*. From the
  resulting page, fill in the relevant slice details, and select the
  *Privileges* tab to define the set of users that are to have access
  to the slice. Alternatively, a site Admin can administer the site's
  slices by selecting the *Slices* tab in the left-hand navigation
  bar, as described in [Getting Started](#getting-started).

* **Nodes:** Select the *Nodes* tab to create and manage the site's
  nodes. To create a new node, select *Add Node*, fill in its FQDN
  and the *Deployment* it belongs to.

* **Deployments:** Select the *Deploymements* tab to affiliate the
  site's nodes with one or more deployments.

## Administering a Deployment

Deployment Admins are responsible for managing the privileges, sites,
images, flavors, and visibility for the deployment. Select the 
*Deployments* tab in the left-hand navigation bar and click on the 
desired deployment. The available deployment details are as follows:

* **Images:** The *Images* selector is used to specify which images 
  are supported by the deployment. Root adminstrators specify the 
  global set of available images.

* **Flavors:** The *Flavors* selector is used to specify which flavors
  are supported by the deployment. Root adminstrators specify the
  global set of available flavors.

* **AccessControl:** The *AccessControl* form is used to specify a
  policy for what users are and are not allowed to access the
  deployment's resources. A given user sees only those deployments
  that have granted access when they attempt to instantiate
  intances. The current policy language is simple: *allow all*
  indicates that all users may instantiate instances on the deployment,
  *allow site &lt;sitename&gt;* grants access to all users from a
  particular site, and *allow user &lt;email&gt;* grants access to a
  specic user.  Deny rules may also be used (*deny site
  &lt;sitename&gt;*, *deny user &lt;email&gt;*, etc).  Rules are
  evaluated in order from top to bottom, with an implicit *deny all*
  at the bottom of the list.

The *Privileges* tab is used to grant other users Admin privileges for
the deployment.

The *Sites* tab is used to bind a Site (and the Nodes they host) to
this Deployment. Nodes are imported into a Deployment through a
*Controller* that is responsible for instantiating and managing the
Nodes. For example, in the case of an OpenStack cluster, the
Controller effectively connects XOS to the Nova, Neutron, and Keystone
services running on the OpenStack head node.

To create a new Deployment-to-Site-to-Controller binding, use the *Add
another Site Deployment* link at the bottom of the Sites tab. You will
be prompted for the Site and the Controller. If an existing Controller
is not suitable, then the "+" button next to the Controller dropdown
may be used to create a new Controller.

Creating a Controller involves defining the following fields:

* **Name:** A human-readable name for the deployment. 

* **Backend Type:** The type of backend for the deployment. Current
    backend types are limited to "OpenStack".

* **Version:** The version of the backend. This is currently limited
    to *Kilo*.

* **Auth URL:** Controller-specific authentication URL. For OpenStack
    controllers, this is the URL of the Keystone endpoint.

* **Admin User:** Controller-specific admin username. 

* **Admin Password:** Controller-specific admin password.

* **Admin Tenant:** Controller-specific admin tenant. 

The list of controllers is not a top-level item in the navigation
panel, so to edit an existing Controller, first go to another object
(e.g., Deployment), and then use the *Core* link at the top of the
page to bring up the list of Core XOS objects. Select the *Change*
link next to *Controller*. *[Workflow to be cleaned up.]*

## Administering Images

To upload an image to XOS, place the image file in `/opt/xos/images`. Note that
adding a file to this directory must be done atomically, for example by uploading
the file elsewhere and then using 'mv' to move the file into the directory. If
a file is uploaded directly to /opt/xos/images, then it's possible the Synchronizer
may attempt to sync the image to controllers while the upload is in progress,
before the file is complete.

Once the image has been placed in /opt/xos/images, the Synchronizer will run and
automatically sync that image to all controllers. 

Even though an image has been uploaded and synced, it is still not available
for use until it has been enabled in a deployment. See the Administering a
deployment section above. 

## Accessing an Administering Services

XOS is designed to support a set of contributed services, but the set
of services available on any particular system is as cloud-specific as
the underlying hardware infrastructure. In the case of OpenCloud,
there are currently two contributed Services, each of which extends
the XOS Data Model with its own abstract objects, and hence, is
accessible via OpenCloud's REST API.

Select the *Services* tab in the left-hand navigation bar to see the
available services. Then select a specific service to either (1) access
or use that service, or (2) administer that service.

Users typically access a service (e.g., mount a Syndicate volume
or publish content in the CDN) through a service-provided view,
but it is also possible to access a service using the *Access* tab.
The details of how the user interacts with the service is beyond
the scope of this Guide. *[Per-service specifications to be added
as an appendix.]*

Service adminstrators typically use the other tabs to manage the
service. In addition to adding a new service, this includes:

* **Details:** The *Details* tab is used to define various XOS-level
parameters for the service, including whether the service is
*Published* (i.e., visible to other users on the *Services* page).

* **Tenancy:** The *Tenancy* tab is used to define dependencies
between this services and other registered services. A collection of
services and service dependencies collectively define the service graph.

* **Slices:** The *Slices* tab is used to define the set of Slices 
that implement (host the instances of) this service.

* **Privileges:** The *Privileges* tab is used give other users
administrative privilege for this service.

* **Attributes:** The *Attributes* tab is used specify attributes
  (key/value pairs) for this service. This provides an easy way to
  prototype extensions to the service without modifying the
  formal data model. (The expectation is that attributes are eventually
  migrated to fields in the service's officially registered data model.)

Information about how to implement a new service is given in
Section [Adding Services](/devguide/addservice/) of the Developer
Guide.


