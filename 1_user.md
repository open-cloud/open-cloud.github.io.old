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

1. **Upload a public key.** Click on the *Users* tab at left, click on
   your email address, and then paste your public key in the box. Click
   the *Save* button at right.

2. **Ask your Site PI to create a slice.** Instructions for PIs (see
     **Information for Site Admins, PIs, and Techs** for more information):
   - Click on the *Slices* tab at left, and then the *Add Slice* button on
     the right. In the *Slice Details* tab, choose a name for the slice
     and select your own site from the drop-down. Slice names must be
     unique. Click the *Save and continue editing* button on the right
     before proceeding.
   - In the *Privileges* tab, click the *Add another slice privilege* link.
     Add the users that will administer the slice with the Admin
     privilege. (These users will be able to extend slice privileges
     to other users.)
   - Click the *Save* button when done.

3. **Create slivers.** Use either the Tenant or the Developer View
   (see next section) to instantiate slivers (VMs) for your slice.

4. **Log into slivers.** Once a sliver comes up, the instance ID will
   appear on the Slivers page for your slice. You should then be able
   to login using this ID and the physical node it is running on.
   Logging into the sliver relies on ssh agent forwarding. You need
   to run ssh-agent on your client, and that your public key is loaded.
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
using the left-hand navigation bar. The top set of items (Deployments,
Sites, Slices, and Users) correspond to the core XOS objects, as
described in **Data Model** Section of the **Overview**. The bottom
set of items (RequestRouter, HyperCache, and Syndicate) correspond to
services that extend XOS. Most users will have no need to directly
access the data model through the navigation bar, instead taking
advantage of the tailoered workflows supported by the various views.

There are also views designed to support operators, including both
thost that operate infrastructure and those that operate
services. Support for operators is given in the **Operator's Guide**.

###Tenant View

The Tenant view provides a simple graphical interface for users to
acquire Slivers, with minimal control over the low-level details of
where those Slivers are placed and how they are interconnected. It is
loosely patterned after the Amazon EC2 interface.

The Tenant view is limited to the ViCCI Deployment, which includes
clusters at five sites throughout the US and Europe.  Users that want
to acquire Slivers on any other deployment must use the Developer
view.

The Tenant view allows users to specify the *Site(s)* on which Slivers
are to be instantiated, select an *Image* and *Flavor* for each
Sliver, mount select *Data Set(s)* into each of the Slice's Slivers,
specify the *TCP Ports* the Slice is going to use, and set the
*ServiceClass* for the Slice (currently only *Best Effort* is
supported).

The *Download Slice Details* button downloads a text file that gives
details about the user's Slice, including DNS names at which the
Slivers can be accessed.  Section **Getting Started** explains how to
use this configuration information to access (e.g., ssh into) the
Slivers instantiated for a Slice.

###Developer View

The Developer view gives users full control over how their Slices are
instantiated, including where *Slivers* are placed, what *Networks*
connect them, and the *Privileges* granted to other users.

With respect to Slivers (configured by selecting the corresponding
tab), users first select a target *Deployment*, and then pick an
individual *Node* at that deployment. Users are also able to select an
*Image* and a *Flavor* on either a per-Sliver or a per-Slice basis. 
All Deployments that the user is permitted to access are visible when
instantiating Slivers.

With respect to Networks (configured by selecting the corresponding
tab), each Slice is automatically configured with a public and a
private network, but the user has the option of connecting the slice's
slivers to additional virtual networks. (Adding additional Networks
to a Slice is currently an undocumented feature.)

With respect to Privileges (configured by selecting the corresonding
tab), the user can grant *Admin* privilege to other users, giving them
the authority to modify slice parameters. Granting a user *Default*
privilege means the corresponding user is granted access to (is able
to ssh into) the slice's slivers.

###xsh View

The xsh view provides an interactive shell through which users can
access XOS objects. It is a Javascript-based environment that includes
*xoslib*, a library projection of the XOS data model. A builtin
tutorial illustrates how to use xsh.

##Information for Site Admins, PIs, and Techs

In addition to creating slices for their site's users (as described in
the *Getting Started* section), there are a few site-related tasks that
site Admins, PIs, and Techs are resposible for.

* **Configuring Site:** ...includes public/private...

* **Approving Users:**

* **Creating Slices:**

* **Adding Nodes:**

* **Assigning Privileges:**

* **Selecting Deployments:**


##Services

OpenCloud includes a set of contributed services that are available to
other OpenCloud users (or the Services running on their behalf). That
is, in some cases end-users are the client/tenant of the Service and
in other cases OpenCloud Services are the client/tenant of the Service.
There are currently three contributed Services, each of which extends
the XOS Data Model with its own abstract objects -- and hence, is
accessible via OpenCloud's REST API -- as follows.

###Syndicate

Syndicate is a scalable storage service for a slice and its users.
When enabled, it provides a private shared volume that both slivers 
and users can locally mount as a read/write filesystem.  In addition,
the OpenCloud deployment of Syndicate offers public read-only volumes
which contain popular scientific datasets for slivers to consume.

####Getting Started

When creating a Slice in the Tenant view, a user can opt to make 
a shared private volume for the slice by checking the "Include 
Private Vol." checkbox.  When the user logs into a sliver, the 
volume will be mounted under `/syndicate/`.

####Developer View

Users can control specific volume parameters from the 
Developer view, under the Syndicate -> Volumes menu.  Users can 
add and remove additional volumes to their slices, bind their volumes 
to other slices, and control volume capabilities for other
OpenCloud users and slices.

**Mounting to an Existing Volume in a Slice**

A user can add another volume to a slice in the following manner
(the user must start from the Volumes menu).

1. Click the desired volume.
2. Select the "Slices" tab.
3. Click "Add another Volume Slice".
4. Fill in the volume parameters, as follows:
    - **Slice**:  Select the slice to receive the volume.
    - **Cap read data**: check this box if a sliver should be
      permitted to read volume data.  If unsure, check this box.
    - **Cap write data**: check this box if a sliver should be
      permitted to write volume data.  If unsure, do not check
      this box.
    - **Cap host data**: check this box if a sliver should be 
      permitted to host data for the volume.  Only check this 
      box if you can rely on the slivers to work with Syndicate
      to keep the volume data consistent and available.  If 
      unsure, do not check this box.
    - **UG port**: This is a port number, between 1024 and 65534,
      on which each sliver's Syndicate User Gateway will listen.
      It does not matter what number the user chooses; it only 
      matters that it is available on each sliver.
    - **RG port**: This is a port number, between 1024 and 65534,
      on which each sliver's Syndicate Replica Gateway will listen.
      It does not matter which number the user chooses; it only 
      matters that it is available on each sliver.
5. Click "Save."

Once these steps are carried out, OpenCloud will ensure that the 
new volume will be automatically mounted under `/syndicate/`
in each sliver.

**NOTE**:  This can only be done with public volumes.  Volumes 
created for a slice in the Tenant view are private by default.
See "Volume Access Control" below.

**Creating and Updating Volumes**

A user can create or change the parameters of their volumes by
selecting their volume from the volume list, and navigating to 
the "Volume Details" tab.  Most of the "Volume Details" fields 
are self-explanatory, but a few warrant additional comment:

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

**Volume Access Control**

Users can authorize other users to access their volumes.  This is not 
limited to within OpenCloud--if Alice permits Bob to access her volume, 
then Bob can do so from his OpenCloud slivers or his personal machines.
Users can control the capabilities other users will have: read access
(*Cap read data*), write access (*Cap write data*), and host access 
(*Cap host data*).

Read and write access is self-explanatory.  Host-access should only 
be granted if the user is sufficiently trustworthy, since the "host data"
capability will grant that user the ability to coordinate writes 
to the volume's files.  If unsure, the user should not check "Cap host data."

###HyperCache

The HyperCache (HPC) Service is used by *Content Providers* to
accelerate the delivery of content to Internet users... [todo]

###RequestRouter

The RequestRouter (RR) Service is used by a client service to redirect
user requests to the best instance of that service. For example,
HyperCache uses RR to redirect an HTTP GET request to the best cache
to serve a particular piece of content to a particular end-user. RR
determines "best" according to the policy specified in a *Service Map*
filed by the client service.

A client (tenant) of the RR Service configures a *ServiceMap* by
specifying the following information: the *Service* requesting the
Service Map; the *Slice* in which the Service runs (which indirectly
identifies the candidate instances to which RR redirects requests);
the URL *Prefix* that end-users will use to name the service; and a
pair of RR-specific map files, called the *SiteMap* and *AccessMap*,
respectively. The format of these two map files is defined elsewhere.

