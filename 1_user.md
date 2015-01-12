---
layout: page
title: User Guide
---

This guide describes how users log into OpenCloud, acquire VMs, and
access OpenCloud services. It assumes the reader is familiar with
the general concepts presented in the **Overview**.

##Getting Started

Users access OpenCloud by logging into the portal at
[opencloud.us](http://opencloud.us). New users register through the
same portal, with the request approved by the PI (Principle
Investigator) at their home site. Sites join OpenCloud through an
off-line process that starts by sending email to info@opencloud.us.

Once the user has an account, the following steps are a quick guide
to utilizing OpenCloud resources:

1. **Upload a public key.** Select the *Users* tab in the left-hand
   navigation bar, click on your email address, and then paste your
   public key in the box. Click the *Save* button at right.

2. **Ask your Site PI to create a slice.** Instructions for PIs (see
     **Administering a Site** for more information):
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

3. **Create slivers.** Use either the Tenant or the Developer View
   (see next section) to instantiate slivers (VMs) for your slice.

4. **Log into slivers.** Once a sliver comes up, the instance ID will
   appear on the Slivers page for your slice, or alternatively, will
   be available to download into a text file. You should then be able
   to login using this ID and the physical node it is running on.
   Logging into the sliver relies on ssh agent forwarding. You need to
   run ssh-agent on your client, and that your public key is loaded.
   If ssh-agent is not already running, you can launch it like so:

```
# ssh-agent bash
# ssh-add ~/.ssh/id_rsa
```

Once ssh-agent is running, you should be able to login to a sliver via
agent forwarding. For example:

```
# ssh -A instance-0000006c@node6.cs.arizona.edu
```

The -A option above is required.

[*A current limitation is that only one user key is injected into the
slice. That user can login and manually add the keys of other users.
We are working on a fix.*]

##Views

Users interact with OpenCloud through a configurable set of *Views*,
each tailored for a different usage scenario or workflow. By default,
a user's home dashboard includes *Tenant*, *Developer*, and *xsh*
views; a fourth tab, *Customize* allows the user to add and remove
views from their home dashboard. The Tenant and Developer views are in
the context of a given (selectable) slice; the xsh view runs in the
context of the user.

Users are also able to directly navigate the underlying data model
using the left-hand navigation bar. The top set of tabs (Deployments,
Sites, Slices, and Users) correspond to the core XOS objects, as
described in **Data Model** Section of the **Overview**. The bottom
set of tabs (RequestRouter, HyperCache, and Syndicate) correspond to
services that extend XOS. Most users will have no need to directly
access the data model through the navigation bar, but will instead
take advantage of the tailoered workflows supported by the various
views.

There are also views designed to support operators, including those
that operate infrastructure and those that operate services. Support
for operators is given in the **Operator's Guide**.

###Tenant View

The Tenant view provides a simple graphical interface for users to
acquire Slivers, with minimal control over the low-level details of
where those Slivers are placed and what networks interconnect them. It
is loosely patterned after the Amazon EC2 interface.

The Tenant view is limited to the ViCCI Deployment, which includes
clusters at five sites throughout the US and Europe. Users that want
to acquire Slivers on any other deployment must use the Developer
view.

The Tenant view allows users to specify the number of slivers that are
to be instantiated on the available sites, select an *Image* and
*Flavor* for each sliver, mount select *Data Set(s)* into each sliver,
and specify the *TCP Ports* the slice is going to use. Users can also
set the *ServiceClass* for the slice, although *Best Effort* is
currently the only supported class.

Click the *Download Slice Details* button to download a text file that
gives details about the slice, including the DNS names at which the
slivers can be accessed. Section **Getting Started** explains how to
use this configuration information to access (e.g., ssh into) the
slivers instantiated for a slice.

###Developer View

The Developer view gives users full control over how their slices are
instantiated, including sliver placement, network configuration, and
the privileges granted to other users:

* **Slivers:** Select the *Slivers* tab at the top to manage the
  slivers bound to the slice. From the resulting page, select a target
  *Deployment*, and then pick an individual *Node* at that deployment. 
  Users are also able to select an *Image* and a *Flavor* on either a
  per-sliver or a per-slice basis. All deployments that the user is
  permitted to access are visible when instantiating slivers.

* **Networks:** Select the *Networks* tab at the top to manage the
  networks connected to the slice. Each slice is automatically
  configured with two virtual networks (one public and one private),
  but the user has the option of connecting the slice's slivers to
  additional virtual networks. (Creating additional networks and
  connecting a slice to them is currently an undocumented feature.)

* **Privileges:** Select the *Privileges* tab at the top to manage the
  users that have access to the slice. Granting a user *Admin*
  privilege means giving them the authority to modify slice
  parameters. Granting a user *Default* privilege means the giving
  them access to the slice itself, that is, the right to ssh into the
  slice's slivers.

###xsh View

The xsh view provides an interactive shell through which users can
access XOS objects. It is a Javascript-based environment that includes
*xoslib*, a library projection of the XOS data model. A builtin
tutorial illustrates how to use xsh.

##Administering a Site

Site Admins are responsible for managing the users, slices, nodes, and
deployments affiliated with the site. Access to information maintained
by XOS for a site can be found by clicking on the *Sites* tab in the
left-hand navigation bar. From there, the Admin can set various site
details, as well as manage various entities using the available tabs:

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
  bar, as described in the **Getting Started** section.

* **Nodes:** Select the *Nodes* tab to create and manage the site's
  nodes. To create a new node, select *Add Node*, fill in...

* **Deployments:** Select the *Deploymements* tab to affiliate the
  site's nodes with one or more deployments....

##Services

OpenCloud includes a set of contributed services that are available to
other OpenCloud users, as well as to services running on their behalf.
In some cases, end-users are the client (tenant) of the service, while
in other cases, services running on OpenCloud are clients of one of
this contributed services. 

There are currently three contributed Services, each of which extends
the XOS Data Model with its own abstract objects, and hence, is
accessible via OpenCloud's REST API.

###Syndicate

Syndicate is a scalable storage service. It provides a private shared
volume that both slivers and users can locally mount as a read/write
filesystem. Syndicate also offers public read-only volumes that
contain popular scientific datasets. Slices can specify that they want
to mount one of these datasets (public volumes).

Support for enabling a private volume and selecting public datasets is
built into the Tenant view, meaning users need not directly access
Syndicate through the corresponding tab in the left-hand navigation
bar. When accessed in this way, the user will see the slice's private
volume mounted under */syndicate* when they log into each of their
slivers.

Selecting the *Syndicate* tab in the left-hand navigation bar and then
clicking on the *Volumes* menu option allows users to control specific
volume parameters. This is a necessary step when using the *Developer*
view. From the resulting page, users can add and remove additional
volumes to their slices, bind their volumes to other slices, and
control volume capabilities for other users and slices.

####Mounting to an Existing Volume in a Slice

A user can add another volume to a slice in the following manner (the
user must start from the Volumes menu).

1. Click the desired volume.

2. Select the *Slices* tab.

3. Click *Add another Volume Slice*.

4. Fill in the volume parameters, as follows:
    - **Slice**: The slice to receive the volume.
    - **Cap read data**: Check this box if a sliver should be
      permitted to read volume data.  If unsure, check this box.
    - **Cap write data**: Check this box if a sliver should be
      permitted to write volume data.  If unsure, do not check
      this box.
    - **Cap host data**: Check this box if a sliver should be 
      permitted to host data for the volume. Only check this 
      box if you can rely on the slivers to work with Syndicate
      to keep the volume data consistent and available.  If 
      unsure, do not check this box.
    - **UG port**: This is a port number, between 1024 and 65534,
      on which each sliver's Syndicate User Gateway will listen.
      It does not matter what number the user chooses; it only 
      matters that it is available on each sliver in the slice.
    - **RG port**: This is a port number, between 1024 and 65534,
      on which each sliver's Syndicate Replica Gateway will listen.
      It does not matter which number the user chooses; it only 
      matters that it is available on each sliver in the slice.
5. Click *Save*.

Once these steps are carried out, Syndicate ensures that the new
volume is automatically mounted under */syndicate/* in each sliver.

####Creating and Updating Volumes

Users can create or change the parameters of their volumes by
selecting their volume from the volume list, and navigating to the
*Volume Details* tab.  Most of the *Volume Details* fields are
self-explanatory, but a few warrant additional comment:

* **Blocksize**:  This is the number of bytes in a block. 
In Syndicate, a block is the smallest unit of transfer, storage, 
and data consistency.  Smaller values reduce the number of 
bytes to invalidate on a write, but larger values improve 
read performance by reducing the number of data transfers.
A good value for read/write volumes is 102400 (100KB).  Read-only
volumes should choose the medium file size as the block size,
to maximize transfer efficiency.

* **Archive**:  If set, this volume will need to be configured 
such that a Syndicate Acquisition Gateway (AG) can populate it 
with dataset metadata.  Do not check this box unless you know 
what you are doing.

* **Cap host data**:  If set, this means that by default, a sliver can 
help Syndicate replicate data and coordinate writes to it.  This 
implies a larger degree of trust in the sliver.  If your slivers 
will do most of the I/O in your volume, you should check this box.
If you or other machines outside of OpenCloud will do most of the I/O,
you should *not* check this box.

####Volume Access Control

Users can authorize other users to access their volumes. This is not
limited to within OpenCloud. If Alice permits Bob to access her
volume, then Bob can do so from his OpenCloud slivers or from his
personal machines. (Likewise, Alice can share her OpenCloud volume
with her own off-OpenCloud machines.)  OpenCloud access to a Users can
control the capabilities other users will have: read access (*Cap read
data*), write access (*Cap write data*), and host access (*Cap host
data*).

Read and write access is self-explanatory. Host-access should only be
granted if the user is sufficiently trustworthy, since the *host data*
capability will grant that user the ability to coordinate writes to
the volume's files.  If unsure, the user should not check *Cap host
data*.

###HyperCache

HyperCache (HPC) is a Content Distribution Nework (CDN). It uses a set
of distributed caches to to accelerate the delivery of content on
behalf of a content provider. Selecting the *HyperCache* tab in the
left-hand navigation bar reveals a menu of options that govern how
content is to be delivered via the CDN. From the resulting pages,
users can add/remove origin servers that source content, register URLs
that name content, and specify where content is (and is not) to be
served.

* **Service Provider:** A CDN administrator. Each service provider
  manages (creates accounts for) a set of content providers.

* **Content Provider:** The primary "tenant" of the CDN. All other
  parameters are controlled by a given content provider. Each content
  provider specifies a set of *Users* that are allowed to act on its
  behalf, and registers one or more *CDN Prefixes* that will name its
  content (see below).

* **CDN Prefix:** The FQDN prefix of URLs that names content to be
  delivered via the CDN. Each CDN Prefix has a default *Origin Server*
  that HPC uses to acquire content (see below).

* **Origin Server:** The URL of an origin server that sources content.
  Each origin server is bound to one CDN Prefix, but one CDN Prefix
  can name content sourced at multiple origin servers. Typically, a
  particular origin server is directly or indirectly identified in the
  URL, but there is an explicitly specified default origin server that
  HPC uses when this is not the case. The content provider also
  specifies the *Protocol* used to download content from an origin
  server (typically, this is set to HTTP), and whether or not requests
  that cannot be handled by the CDN should be redirected to the origin
  server (typically, this setting is enabled).

* **Site Map:** A file that specifies a policy for how requests are to
    be redirected.

* **Access Map:** A file that specifies a policy for where content is
    and is not delivered.

The format of the map files is defined elsewhere.

###RequestRouter

RequestRouter (RR) is a multi-tenant service that redirects user
requests to the best instance of a given client service. For example,
HyperCache uses RR to redirect HTTP GET requests to the best cache to
serve content to a particular end-user. RR determines "best" according
to a client-specified policy known as a service map.

Selecting the *RequestRouter* tab in the left-hand navigation bar,
clicking the *Service Map* menu item, and then selecting a particular
map allows users to specify the policy that guides request redirection.
From the resulting page, users can register a client service, register
a URL that name the service, and configure various maps.

* **Name:** Name of the service map.

* **Owner:** Client service that owns the service map.

* **Slice:** Slice in which the client service runs. Indirectly
  identifies the candidate service instances to which RR redirects
  requests.

* **Prefix:** The FQDN prefix of a URI that is to be managed by
    RequestRouter. This prefix effectively names the service in a
    server-independent way.

* **Site Map:** A file that specifies a policy for how requests are to
    be redirected.

* **Access Map:** A file that specifies a policy for where the service
    is and is not available.

The format of the two map files is defined elsewhere.

