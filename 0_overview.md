---
layout: page
title: Overview
---

OpenCloud is an operational cloud running on servers distributed
across multiple sites world-wide. This section describes both its
software architecture and the hardware infrastructure on which it is
currently deployed.

To help frame the discussion, this guide distinguishes among three
independent ideas:

* *Everything-as-a-Service (XaaS)* is the organizing principle that
  underlyies OpenCloud's design. We assume the reader is familiar with
  this general concept. See the relevant
  [whitepapers](http://opencloud.us/whitepapers.html) for more information.

* *XaaS Operating System (XOS)* is a Cloud OS that that implements the
  XaaS architecture. XOS runs on OpenCloud, but is also available as
  open source software. Developers can access XOS source code on
  [GitHub](http://open-cloud.github.io).

* *OpenCloud* is an operational cloud that runs XOS and spans a set of
  servers distributed across multiple sites world-wide. Users can
  access OpenCloud resources via the portal at
  [opencloud.us](http://opencloud.us).

This guide primarily focuses on XOS, although examples from OpenCloud
are used to illustrate operational and deployment aspects.

##Software Architecture
---

Figure 1 schematically depicts the software components running on
OpenCloud.  The bottom-most green boxes represent low-level
virtualization mechanisms, including Open vSwitch (OvS), libvirt (an
interface to LXC and KVM), and OpenVirteX (a Network Hypervisor that
dynamically creates customizable virtual networks on top of OpenFlow
switches). A detailed description of OpenVirteX can be found in a
companion [whitepaper](http://opencloud.us/docs/OpenVirteX.pdf).

![Figure 1. Software components running on OpenCloud.]({{ site.url }}/figures/Slide1.jpg)

Figure 1. Software components running on OpenCloud.

The middle orange boxes represent a collection of services, or more
specifically, the controllers for those services. This includes a set
of core services -- Compute-as-a-Service (CaaS), Network-as-a-Service
(NaaS), and Id-Management-as-a-Service (IDaaS) -- required to "boot"
the system, as well as library of contributed services. The core
services are borrowed from OpenStack, and collectively implement
Infrastructure-as-a-Service (IaaS). The contributed services are drawn
from a combination of research prototypes, open source projects, and
trial deployments of commercial services.

The top-most purple box represents XOS proper. By way of analogy, XOS
corresponds to the Unix kernel, where the set of services built around
XOS correspond to the commands-and-libraries bundled with Unix. An
important aspect of this design is that to most users, the distinction
between the kernel and the commands-and-libraries is not important.
That is, XOS includes explicit support for folding new services built
on top of OpenCloud back into OpenCloud, making them available to
users. 

More information about how users interact with XOS is given in the
**User Guide**, while more information about how to extend XOS with
new services is given in the **Developer Guide.** XOS also supports
cloud operators that manage physical resources (i.e., control the
hardware, establish resource allocation policies, select
virtualization technologies), and it does so in a way that supports
multiple administrative domains. More information about how operators
use XOS to manage a cloud is given in the **Operator Guide**.

![Figure 2. Internal structure of XOS influenced by MVC pattern.]({{ site.url }}/figures/Slide2.jpg)

Figure 2. Internal structure of XOS influenced by MVC pattern.

Figure 2 depicts the internal structure of XOS, which is borrows
heavily from the Model-View-Controller (MVC) design pattern originally
introduced by Smalltalk, but now considered best practice because of
its clean separation of data representation and presentation. The
*Data Model* codifies the abstract objects, the relationship among
those objects, and the operations on those objects. It is the heart of
XOS, and so we briefly introduce the key object types in the next
section.

On top of these abstractions is a set of *Views* that defines how
various actors interact with XOS. These include traditional cloud
tenants, service developers, and cloud operators. We expect the set of
views to evolve over time, so for example, if OpenCloud one day
supports a suite of content distribution services, it might be useful
to create a "Content Provider" view.

Below the Model sits a collection of controller plugins that react to
changes in the Data Model by manipulating the interfaces to the
constituent services. We call this set of plugins -- and the framework
in which they execute -- the *Observer*. The Observer's job is to
ensure configuration state remains consistent across all levels of the
distributed system, from the authoritative state maintained in the XOS
data base, to the per-service state maintained in each service
controller, to the local state in the distributed instances that
implement each service. This is a challenging technical problem, and
our approach is informed by a decade of experience controlling
PlanetLab nodes and slices distributed across hundreds of sites around
the world. A companion
[whitepaper](http://opencloud.us/docs/Observer.pdf) describes the
solution adopted by the Observer in more detail.

XOS is designed to be extensible, as described in the **Developer
Guide**: Section **Adding Views** describes how to extend XOS to
include new views (it involves writing "applications" on top of the
XOS data model), and Section **Adding Services** describes how to
extend XOS to include new services (it involves adding controller
plugins to the Observer).

##Data Model
---

This section gives a high-level overview of the XOS data model. This
overview focuses on the abstract objects and relationships among them,
and should not be read as an formal specification. A detailed and
up-to-date definition is included in the Appendix.

The data model is implemented in Django, so we (mostly) adopt Django
terminology to describe it. Briefly, what is often referred to as an
*Object Class* or *Object Type* is called a *Content Type* or *Model*
in Django. This document uses the term *Object Type* for this concept.

An Object Type defines a set of *Fields*, each Field has a *Type*, and
each Type has a set of *Attributes*. Some of these Attributes are core
(common across all Types) and some are Type-specific. Relationships
between Object Types are expressed by Fields with one of a set of
distinguished relationship-oriented Types (e.g., *OneToOneField*). 
Finally, an *Object* is an instance (instantiation) of an Object Type,
where each Object has a unique primary key (or more precisely, a
primary index *Object Id* into the table that implements the Object
Type).

The following introduces and motivates XOS's core Object Types, along
with some of their key Fields, including relationships to other Object
Types. The discussion is organized around six categories: access
control, infrastructure, policy, virtualization, accounting, and
services.

###Access Control

XOS uses role-based access control, where a user with a particular
role is granted privileges in the context (scope) of some set of
objects.

* **User:** A principle that invokes operations on objects and uses
  XOS resources.

* **Role:** A set of privileges granted to a User in the context of
  some set of objects.

* **RootPrivilege:** A binding of a User to a Role in the context of
  all objects. Root-level roles include:

  - **Admin:** Read/write access to all objects.

  - **Default:** Read/write access to this User; read-only access to
    Users, Sites, Deployments, and Nodes.

* **SitePrivilege:** A binding of a User to a Role in the context of a
  particular Site, which implies the Role applies to all Nodes,
  Slices, Invoices, and Users associated with the Site. Site-level
  roles include:

  - **Admin:** Read/write access to all Site-specific objects.

  - **PI:** Read/write access to a Site's Slices and Users.

  - **Tech:** Read/write access to a Site's Nodes.

  - **Billing:** Read/write access to a Site's Account and Invoices.

  - **Default:** Read-only access to all of a Site's objects.

* **SlicePrivilege:** The binding of a User to a Role in the context
  of a particular Slice, which implies the Role applies to all Slivers
  and Networks associated with the Slice. Slice-level roles include:

  - **Admin:** Read/write access to all Slice-specific objects.

  - **Default:** Allowed to access Slivers instantiated as part of the
    slice.

* **Deployment Privileges:** The binding of a User to a Role in the
  context of a particular Deployment, which implies the Role applies
  to all Objects of type Image, NetworkTemplate, and Flavors
  assocaited with the Deployment. Deployment-level roles include:

  - **Admin:** Read/write access to all Deployment-specific objects.

  - **Default:** Read-only access to all Deployment-specific objects.

Operationally, Root-level Admins create Sites, Deployments, and Users,
granting Admin privileges to select Users affiliated with Sites and
Deployments. By default, all Users have read-only access to all Sites,
Deployments, and Nodes, and read/write access to its own User Object.

Site Admins create Nodes, Users, and Slices associated with the Site. 
The Site Admin may also grant PI privileges to select Users (giving
them the ability to manage the Site's Users and Slices), and Tech
privileges to select Users (giving them the ability to manage the
Site's Nodes). By default, all Users affiliated with a Site have
read-only access to the Site's objects.

Site Admins and PIs grant Admin privileges to select Users affiliated
with the Site's Slices. These Users, in turn, add additional Users to
the Slice and instantiate the Slice's Sliver's and Networks. By
default, all Users affiliated with a Slice have read-only access to
the Slice object, and may access (ssh into) the Slice's Slivers.

Deployment Admins manage Deployments, including defining their
operating parameters, specifying what Sites may contribute Nodes to
the Deployment, and specifying what Sites may acquire resources from
the Deployment. By default, all Users affiliated with a Deployment
have read-only access to to the Deployment's objects.

Note that while the data model permits many different bindings between
objects, all of the above scoping rules refer to a parent/child
relationship. For example, a User can be granted a Role at multiple
Sites, Slices, and Deployments, but it is homed at (managed by)
exactly one Site.

Also, the Admin privilege is always scoped at the level to which it
has been assigned, and allows that user to grant privileges to any
user/object combinations that they themselves are privileged for. For
example, if user Jane.Smith has the Admin SitePrivilege for Princeton
and Stanford, then she may assign a Role to a user at Princeton for
default access to Stanford.

###Infrastructure

XOS manages of a set of physical servers deployed throughout the
network. (See Section **Deployment** for a description of the
infrastructure that makes up OpenCloud.) These servers are aggregated
along two dimensions, one related to location and the other related to
policy. This approach is motivated by the need to decouple the hosting
organization from the operational organization, both of which
collaborate to manage nodes. This results in three core Object Types:

* **Nodes:** A physical server that can be virtualized.

  - Bound to one Site that hosts the Node and shares responsibility
    for operating it.

  - Bound to one Deployment that defines the policies applied to the
    Node.

* **Site:** A logical grouping of Nodes that are co-located at the
  same geographic location, which also typically corresponds to the
  Nodes' location in the physical network.

  - Bound to a set of Users that are affiliated with the Site.

  - Bound to a set of Nodes located at the Site.

  - Bound to a set of Deployments that the Site may access.

  - Bound to an Account that reports the Site's resource consumption.

* **Deployment:** A logical grouping of Nodes running a compatible set
  of virtualization technologies and being managed according to a
  coherent set of resource allocation policies.

  - Bound to a set of Users that establish the Deployment's
    policies.

  - Bound to a set of Nodes that adhere to the Deployment's policies.

  - Bound to a set of supported Images that can be booted on the
    Deployment's nodes.

  - Bound to a set of supported NetworkTemplates that can connect VMs
    in the deployment.

  - Bound to a set of supported Flavors that establish allocation
    parameters.

Sites and Deployments can be one-to-one, which corresponds to a each
Site establishing its own policies. In practice, however, we expect
Deployments will often span multiple Sites, where those Sites either
correspond to a single distributed organization (e.g., Internet2) or
agree to manage their Nodes in collaboration with the containing
Deployment (e.g., Enterprise). Although not currently exercised in
OpenCloud, it is also possible that a Site hosts Nodes that belong to
more than one Deployment.

Operationally, the Root-level Admin creates Sites and Deployments. The
Site-level Admin and Tech create and manage Nodes at that Site, and
binds each of the Site's Nodes to one of the available Deployments.
Each Deployment-level Admin sets the policy and configuration
parameters for the Deployment, decides what Sites are allowed to host
Nodes in that Deployment, and decides what Sites are allowed to
instantiate Slivers and Networks on the Deployment's resources.

###Policies and Configurations

Each Deployment defines a set of parameters, configurations and
policies that govern how a collection of resources are managed. XOS
models these as follows:

* **Image:** A bootable image that runs in a virtual machine. Each
  Image implies a virtualization layer (e.g., LXC, KVM), so the latter
  need not be a distinct object.

* **Flavor:** A pre-packaged collection of resources (e.g., disk,
  memory, and cores). Current flavors borrow from EC2.

* **NetworkTemplate:** A loadable specification of a virtual
  network. Each NetworkTemplate implies a virtualization layer (e.g.,
  Neutron's default plug-in), so the latter need not be a distinct
  object. A NetworkTemplate can be parameterized.

Each Deployment defines the set of Images and NetworkTemplates it
supports -- and by implication, the virtualization layers it supports
-- along with the configuration parameters that govern the available
Flavors (e.g., limits on the number of cores that can be allocated to
a given Slice during some unit of time).

We expect there to be a small number of system-supported
virtualization layers for both virtual machines and virtual
networks. We also expect there to be a set of canned Images and
NetworkTemplates available for use, where the system provides a means
to upload custom Images and NetworkTemplates. (Slices can also
parameterize an existing NetworkTemplate.) Note that Images and
NetworkTemplates are analogous constructs in the sense that both are
opaque objects from the perspective of the data model.

###Virtualization

A virtualized Slice of the physical infrastructure is allocated and
managed as follows:

* **Slice:** A resource container that includes the compute and
  network resources that belong to (are used by) a set of users to run
  some distributed service or cloud application.

  - Bound to a set of Users that manage and use the Slice's
    resources.

  - Bound to a (possibly empty) set of Slivers that instantiate the
    Slice.

  - Bound to a set of Networks that connect the Slice's Slivers.

  - Bound to a Flavor that defines how the Slice's Slivers are
    scheduled.

  - Bound to a Image that boots in each of the Slice's Slivers.

  - Bound to a Usage object that records data about resource
    consumption.

  - Optionally bound to a Service that defines the Slice's interface.

* **Sliver:** A single instance (VM) associated with a Slice. Each
  Sliver is instantiated on some physical Node.

* **Usage:** Records Slice-specific state about resource usage.

* **Network:** A virtual network associated with a Slice. Each Network
 also specifies a (possibly empty) set of other Slices that may join
 it, has an IP AddrSpace (CIDR block and port range) by which all
 Slivers are addressed, is designated at Public or Private depending
 on whether its addresses are publicly routable, has a
 NetworkParameter that parameterizes the selected NetworkTemplate, and
 has a set of application-specific Labels that indicates how the
 Network is to be used.

  - Bound to a single "owner" Slice.

  - Bound to a (possibly empty) set of "participating" Slices.

  - Bound to a NetworkTemplate that defines its behavior.

  - Bound to (optionally) one Site, for site-specific networks that
    are part of a Deployment's infrastructure.

Operationally, once an Admin or PI at a Site has created a Slice and
assigned an initial set of Users (with Admin privilege) to that Slice,
those Users instantiate the Slice on the underlying infrastructure by
creating a set of Slivers and a set of Networks. Optionally, a Slice
may join Networks created (owned) by other Slices. Note that the set
of Slivers and Networks instantiated for a Slice may span multiple
Deployments, but this fact is not necessarily visible to the User.

The workflow for instantiating a Slice on the physical infrastructure
is typically iterative and incremental. A User associated with the
Slice might first query the system to learn about the set of Sites and
Nodes and create a set of Slivers accordingly. Next, the User might
create one or more Networks that logically connect those Slivers. 
Slivers can be added to and removed from a Slice over time, with the
corresponding Networks adjusted to account for those changes.

The system maintains the invariant that all Slivers belonging to the
Slice are attached to all Networks associated with the Slice, with the
convention that every Network appears as an interface (in the sense of
an Unix interface, for example, eth1) within each Sliver. Each such
interface inherits the Labels associated with the Network, such that
the program running in the Sliver can query the interface for these
Labels to learn how the corresponding Network is being used.

There is intended asymmetry in the definition of a Slice. All Slivers
bound to a Slice share the same Image (hence that field is defined
Slice-wide), while each Network potentially has a different
NetworkTemplate (hence that field is defined per-Network).

###Billing and Accounting

XOS includes a billing model that can be used to charge Sites for
resource usage. The model includes enough detail so the Site can pass
charges on to the responsible Slice. Includes the following objects:

* **Account:** Record of billing information (e.g., contacts) for a
    Site.

  - Bound to a set of Invoices.

* **Invoice:** Charges for the resources consumed by all Slices at
  this Site over some time period (e.g., weekly).

  - Bound to a set of Charges that aggregate charges over some time
    interval.

* **Charge:** Detailed report of resource usage at the granularity of
  a Slice.

  - Bound to a Slice-specific Usage object.

Operationally, a User with the (Site, Billing) Role maintains the
Account for the Site, and receives periodic email notifications of
Invoices that have been generated by the system. XOS's underlying
resource allocation mechanism generates and maintains a set of Invoice
objects, each of which aggregates a set of Charges over some time
interval. Each Charge includes a reference to a per-Slice Usage
object.

###Services

XOS goes beyond Slices to define a model for the service runningwithin a Slice:

* **Service:** A registered network service that other Services can
  access via extensions to the XOS data model and API. Each Service is

  - Bound to a set of Slices that collectively implement the Service.

Operationally, service developers -- Users with Admin privileges for
the Slice(s) that implement a Service -- create Service objects, which
automatically registers them with XOS. Doing so may also involve
defining one or more Service-specific objects that extend the core
data model, and provide other users with a means to create an
"instance" of the Service for their use (e.g., create a "Volume" for
storage service). Extending XOS with these new Service-specific
objects is not currently a first-class operation in XOS, but rather,
involves directly augmenting the data model, as described in the
**Adding Services** section of the **Developer Guide**.

##Interfaces
---

XOS offers two layered interfaces. The primary interface is a RESTful
API running directly on top of Django. The second is in the form of a
library, called *xoslib*, that simplifies the task of building Views.

##Hardware Infrastructure
---

This section sketches the OpenCloud hardware infrastructure, both in
general terms (i.e., what OpenCloud will look like when fully
deployed), and in specific terms (i.e., what infrastructure is
operational today).

OpenCloud is designed to span a wide spectrum of Cloud resources --
from the data center, across wide-area national and regional networks,
to edge access networks -- with SDN-enabled networks providing
end-to-end connectivity across the entire system.

![Figure 3. OpenCloud Deployed Across Four Tiers of Resources.]({{ site.url }}/figures/Slide3.jpg)

Figure 3: OpenCloud Deployed Across Four Tiers of Resources.

Figure 3 illustrates the scope of OpenCloud's proposed reach across
four tiers of resource providers. At the far left are commodity clouds
like Amazon's EC2 and Rackspace; next are data center clusters
(provided by CloudLab and ViCCI); then come medium-sized clusters
embedded in Internet2 routing centers and regional PoPs (ViNI);
finally, many small clusters are located at the network edge and on
campuses, close to end-users (on-campus private clouds).

At all four tiers, integrating a given provider's resources into
OpenCloud involves extending XOS to invoke the underlying service
controller(s) exported by that that provider. Abstractly, OpenCloud is
a single virtual cloud that spans a set of underlying cloud providers,
while in practice, XOS provides a uniform software layer that
integrates a set of underlying service controllers. For providers that
offer bare metal (all but the left-most commodity cloud providers),
OpenCloud also installs and operationalize OpenStack on the available
servers. The following discusses four subcases in more detail.
    
First, OpenCloud consolidates and repurposes hardware already being
used existing research testbeds. This includes the five mid-sized
ViCCI clusters running at Princeton, Stanford, Washington, Georgia
Tech, and the Max Planck Institute (each cluster has 70 servers) and
the ViNI servers deployed across ten Internet2 routing centers (each
ViNI site currently hosts 2-4 high-end servers). OpenCloud will
eventually leverage InstaGENI servers being deployed at 32 campuses in
the U.S. Each GENI cluster includes 6 servers, 2-4 of which are
currently bootable as OpenCloud nodes.
    
Second, OpenCloud integrates commodity clouds, most notably Amazon's
EC2. This is possible because XOS treats everything-as-a-service,
including the VM provisioning service offered by commodity cloud
providers (specifically, EC2's programmatic interface). These public
resources do not allow the same low-level control over the underlying
infrastructure as XOS enables, but for those cases where commodity VMs
are sufficient, integrating access to such resources into OpenCloud
simplifies the process of incorporating them into a given slice.
    
Third, OpenCloud will eventually run on CloudLab, which includes
large clusters at three US sites: Clemson, Wisconsin, and Utah. Our
expectation is that the allocation will be modest at first, and that
over time, the number of servers allocated to OpenCloud will vary
based on demand.

Fourth, OpenCloud will take advantage of SDN-enabled networks
throughout this infrastructure. Each cluster includes OpenFlow-enabled
top-of-rack switches. The ViNI servers throughout Internet2 are
co-located with OpenFlow switches deployed as part of the Network
Development and Deployment Initiative (NDDI). Due to NSF's Campus
Cyperinfrastructure -- Network and Infrastructure and Engineering
(CC-NIE) program, OpenFlow switches are also now beginning to be
deployed throughout dozens of campus and regional networks.
    
Today, OpenCloud includes servers at all four tiers, organized as four
independently managed Deployments: an EC2 Deployment allows users to
acquire VMs in EC2, a ViCCI Deployment includes six servers at each of
the five ViCCI sites (ViCCI servers will be as demand dictates), an
Internet2 Deployment includes two servers at each of ten Internet2
routing centers, and an Enterprise Deployment includes clusters at two
University campuses.

