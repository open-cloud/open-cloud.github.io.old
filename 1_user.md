---
layout: page
title: User Guide
---

##Getting Started

Users access OpenCloud by logging into the portal at
[opencloud.us](http://opencloud.us). New users register through the
same portal, with the request approved by the PI (Principle
Investigator) at their home site. Sites join OpenCloud through an
off-line process that starts by sending email to info@opencloud.us.

Once the user has an account, the following steps are a quick guide
to utilizing OpenCloud resources:

1. **Upload a public key.** Click on the Users tab at left, click on
   your email address, and then paste your public key in the box. Click
   the Save button at right.

2. **Ask your Site PI to create a slice.** Instructions For PIs:

   - Click on the Slices tab at left, and then the Add Slice button on
     the right. In the Slice Details tab, choose a name for the slice
     and select your own site from the drop-down. Slice names must be
     unique. Click the Save and continue editing button on the right
     before proceeding.

   - In the Privileges tab, click the Add another slice privilege link.
     Add the users that will administer the slice with the Admin
     privilege. (These users will be able to extend slice privileges
     to other users.)

   - Click the Save button when done.

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

Note: The -A option above is required.

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
described in Data Model Section of the Overview. The bottom set of
items (RequestRouter, HyperCache, and Syndicate) correspond to
services that extend XOS. Most users will have no need to directly
access the data model through the navigation bar, instead taking
advantage of the tailoered workflows supported by the various views.

There are also views designed to support operators, including both
thost that operate infrastructure and those that operate
services. Support for operators is given in the *Operator's Guide*.

###Tenant View

The Tenant view provides a simple graphical interface for users to
acquire VMs, with minimal control over the low-level details of where
those VMs are placed and how they are interconnected. It is loosely
patterned after the Amazon EC2 interface. 

The Tenant view is limited to the ViCCI Deployment, which includes
modest-sized clusters at five sites throughout the US and Europe.
Users that want to acquiring VMs on any other deployment must use
the Developer view.

The Tenant view supports two modes: Simple (the default) and Advanced
(accessed by selecting the *Go To Advanced View* button). Simple mode
lets users specify how many VMs (Slivers) they want. The VMs are
placed at the best available site and boot a default image. Advanced
mode lets the user specify the site(s) on which VMs are to be
instantiated, select a Image to boot in those VMs, select a Service
Level, mount select Data Set(s) into those VMs, and specify the TCP
ports the slice is going to use.

The *Download Slice Details* button available in both Simple and
Advanced mode downloads a text file that gives details about the
user's slice, including DNS names at which the VMs can be accessed.
Section *Accessing a Slice* explains how to use this configuration
information to access (e.g., ssh into) the Slivers instantiated for a
Slice.

###Developer View

The Developer view gives users full control over how their slices are
instantiated, including where *Slivers* are placed, what *Networks*
connect them, and the *Privileges* granted to other users.

With respect to Slivers (configured by selecting the corresponding
tab), users first select a target *Deployment*, and then pick an
individual *Node* at that deployment. Users are also able to select an
*Image* and a *Flavor* on a per-Sliver basis. All Deployments that the
user is permitted to access are visible when instantiating Slivers.

With respect to Networks (configured by selecting the corresponding
tab), each slice is automatically configured with a public and a
private network, but the user has the option of connecting the slice's
slivers to additional virtual networks.

With respect to Privileges (configured by selecting the corresonding
tab), the user can grant *Admin* privilege to other users, giving them
the authority to modify slice parameters. Granting a user *Default*
privilege means the corresponding user is granted access to (is able
to ssh into) the slice's slivers.

###xsh View

The xsh view provides an interactive shell through which users can
access XOS objects. It is a Javascript-based environment that includes
xoslib, a library projection of the XOS data model. A builtin tutorial
illustrates how to use xsh.

##Services

###Syndicate

###HyperCache

###RequestRouter

