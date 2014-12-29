---
layout: page
title: User Guide
---

##Accessing an Account

Users access OpenCloud by logging into the portal at
[opencloud.us](http://opencloud.us). New users register through the
same portal, with the request approved by the PI (Principle
Investigator) at their home site. Sites join OpenCloud through an
off-line process that starts by sending email to info@opencloud.us.

##Views

Users interact with OpenCloud through a configurable set of *Views*,
each tailored for a different usage scenario or workflow. By default,
a user's home dashboard includes *Tenant* and *Developer* views; a
third tab, *Customize* allows the user to add and remove views from
their home dashboard. User views are in the context of a given
(selectable) slice. 

Users are also able to directly navigate the underlying data model
using the left-hand navigation bar. The top set of items (Deployments,
Sites, Slices, and Users) correspond to the core XOS objects, as
described in Data Model Section of the Overview. The bottom set of
items (RequestRouter, HyperCache, and Syndicate) correspond to
services that extend XOS. Most users will have no need to directly
access the data model through the navigation bar, insteading opting to
take advantage of the tailoered workflows supported by the various
views.

There are also views designed to support operators, including users
that operate infrastructure and users that operate services. Support
for such users is given in the *Operator's Guide*.

###Tenant

The Tenant view provides a simple graphical interface for users to
acquire VMs, with minimal control over the low-level details of where
those VMs are placed and how they are interconnected. It is loosely
patterned after the Amazon EC2 interface.

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

###Developer

The Developer view gives users full control over how their slices are
instantiated, including where *Slivers* are placed, what *Networks*
connect them, and the *Privileges* granted to other users.

With respect to Slivers (configured by selecting the corresponding
tab), users first select a target *Deployment*, and then pick an
individual *Node* at that deployment. Users are also able to select
an *Image* and a *Flavor* on a per-Sliver basis.

With respect to Networks (configured by selecting the corresponding
tab), each slice is automatically configured with a public and a
private network, but the user has the option of connecting the slice's
slivers to additional virtual networks.

With respect to Privileges (configured by selecting the corresonding
tab), the user can grant *Admin* privilege to other users, giving them
the authority to modify slice parameters. Granting a user *Default*
privilege means the corresponding user is granted access to (is able
to ssh into) the slice's slivers.

###xsh

##Services

###Syndicate

###HyperCache

###RequestRouter

##Accessing a Slice



