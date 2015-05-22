---
layout: page
title: Architecture Guide
---

XOS is an extensible cloud operating system that defines unifying
abstractions on top of a collection of cloud services. We call XOS an
operating system because it plays much the same role in a
geo-distributed cloud as a traditional operating system does on a
conventional computer -- it provides general programming abstractions
to support a wide range of applications, while at the same time safely
multiplexing the underlying hardware resources and software services
between them. By regarding XOS as an operating system, we bring 50
years of proven concepts and best practices to the problem of
tranforming the cloud into a general-purpose computing environment.

XOS leverages existing datacenter cloud management systems (e.g.,
OpenStack) to implement low-level scheduling and resource allocation
mechanisms for each autonomous site, and SDN-based network
customization (e.g., control functions running on top of ONOS) to
connect the sites.

XOS provides explicit support for multi-tenant services, making it
possible to create, name, operationalize, manage and compose services
as first-class operations. Well-known multi-tenant clouds are designed
to host applications, but they usually treat these as single-tenant
services that run on top of the cloud, for the benefit of the user. In
contrast, XOS provides a framework for implementing multi-tenant
services that become part of the cloud, thereby lowering the barrier
for services to build on each other.

##OS Perspective

XOS is designed from the perspective of an operating system. An OS
provides many inter-related mechanisms to empower users. If we define
an OS by a successful example, Unix, then an OS provides isolated
resource containers in which programs run (e.g., processes);
mechanisms for programs to communicate with each other (e.g., pipes);
conventions about how programs are named (e.g., /usr/bin), configured
(e.g., /etc), and started (e.g., init); a mechanism to program new
functionality through the composition of existing programs (e.g.,
shell); a means to identify and authorize principles (e.g., users);
and a means to incorporate new hardware into the system (e.g., device
drivers).

XOS provides counterparts to all these mechanism, but in general
terms, XOS adopts much the same design philosophy as Unix. Both are
organized around a single cohesive idea: everything is a file in Unix
and *Everything-as-a-Service (XaaS)* in XOS.  Both also aim to have a
minimal core (kernel) and are easily extended to include new
functionality. In Unix, the set of extensions correspond to the
applications running top of the Unix kernel. This is also the case
with XOS, where as depicted in Figure~1, the core is minimal and the
interesting functionality is provided by a collection of services. 
Moreover, XOS supports a shell-like mechanism that makes it possible
to program new functionality from a combination of existing services.

![Figure 1. XOS layers an OS on top of a set of cloud services (service controllers).]({{ site.url }}/figures/slide18.jpg)

Figure 1. XOS layers an OS on top of a set of cloud services (service controllers).

Implicit in Figure~1 is an underlying model of exactly what
constitutes a service. Our model is that every service incorporated
into XOS provides a logically centralized interface, called a *service
controller*; elastically scales across a set of *service instances*;
and is multi-tenant with an associated *tenant abstraction*. Figure~2
depicts the anatomy of a service in this way, where the instances (for
example, VMs) are potentially distributed widely over a
geo-distributed set of clusters. For example, some VMs might be
concentrated in one or more datacenters, while others are distributed
across many edge sites.

![Figure 2. Anantomy of a Service.]({{ site.url }}/figures/slide22.jpg)

Figure 2. Anatomy of a Service.

The separation of service controller from service instances is central
to XOS's design. The controller maintains all authoritative state for
the service, and is responsible for configuring the underlying
instances. Service users (and service operators) interact with the
controller, which exposes a global interface; any per-instance or
per-device interface is an implementation detail that is hidden behind
the controller.

##Software Structure

The XOS implementation is organized around three layers, as
illustrated in Figure~3. At the core is a *Data Model*, which records
the logically centralized state of the system. It is the Data Model
that ties all of the services together, and enables them to
interoperate reliably and efficiently.  The logical centralization of
this state is achieved through a clearcut separation between this
*authoritative* state and the ongoing, fluctuating, and sometimes
erroneous state of the remainder of the system: the so-called
*operational* state. The ability to distinguish between the overall
state of the system at these two levels (authoritative Data Model and
operational backend) is a distinguishing property of XOS.

![Figure 3. Block diagram of the XOS software structure.]({{ site.url }}/figures/slide21.jpg)

Figure 3. Block diagram of the XOS software structure.

The Data Model encapsulates the abstract objects, relationships among
those objects, and operations on those objects. The operations are
exported as a RESTful HTTP interface, as well as via a library
(*xoslib*) that provides a higher-level programming interface. On top
of this Data Model, a set of *Views* defines the lens through which
different users interact with XOS. For example, Figure~3 shows a view
tailored for tenants, one tailored for service developers, and one
customized for service operators. Finally, a *Controller Framework* is
responsible for distributed state management; that is, keeping the
state represented by a distributed set of service controllers in sync
with the authoritative state maintained by the Data Model.

The Controller Framework is a critical component of XOS. It binds the
logically centralized authoritative state to the rest of the system,
synchronizes the policies specified in its upper levels to it
lower-level mechanisms, and keeps the policies themselves consistent.
Instead of transmitting changes in the authoritative state in the form
of deltas sent out to various service controllers, it computes these
deltas from the service controller's vantage point. With this
strategy, the authoritative state of the system can be determined
unambiguously at any given time, even if the operational state of the
rest of the system is lagging behind, or is even erroneous.

Furthermore, any consistency properties encoded as policies in the
form as functions of the state can be applied to the central state
independent of the service controllers. This decoupling of managing
the central state and orchestrating backend mechanisms is instrumental
in enabling the logical centralization of distributed resources. And
the resulting parallel construct is not an accident. Just as
individual services have a logically centralized controller and a
scalable set of instances, XOS is structured to provide a logically
centralized interface on top of a cloud-wide set of services.

The current implementation of XOS uses a combination of open source
software and commodity services. The Data Model is implemented in
Django, and leverages a substantial Django ecosystem (e.g., Django
Suit is used to automatically generate an admin GUI). Views are
Javascript programs running on the user's browser, where xoslib is a
client/server library that uses Backbone.js and Marionette.js over the
HTTP-based REST API. The Controller Framework is a from-scratch
program (called the Observer) for executing service controller
plug-ins. It leverages Ansible to handle low-level configuration with
the back-end controllers.

XOS runs on top of a mix of service controllers, some instantiated as
part of an XOS deployment, and some made available by commodity
providers. Today, these include OpenStack (Nova, Neutron, Keystone,
Ceilometer, and Glance), EC2, HyperCache and RequestRouter
(proprietary CDN services from Akamai), ONOS and OpenVirtex (a network
operating system and network hypervisor, respectively), and Syndicate
(a research storage service built on top of S3, Dropbox, HyperCache,
and Google App Engine). We have also prototyped multi-tenant services
using several open source projects, including Cassandra, Kairos,
Swift, and Nagios.

##<a name="data-model">Data Model</a>

This section gives a high-level overview of the XOS data model. This
overview focuses on the abstract objects and relationships among them,
and should not be read as a formal specification. A detailed and
up-to-date specification of the REST API exported by the data model is
available at
[portal.opencloud.us/docs/](http://portal.opencloud.us/docs/).

The data model is implemented in Django, so we (mostly) adopt Django
terminology to describe it. Briefly, what is often referred to as an
*Object Class* or *Object Type* is called a *Content Type* or *Model*
in Django. This guide uses the term *Object Type* for this concept.

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
Types. The discussion is organized around five categories: access
control, infrastructure, policy, virtualization, and services.

###Access Control

XOS uses role-based access control, where a user with a particular
role is granted privileges in the context (scope) of some set of
objects.

* **User:** A principle that invokes operations on objects and uses
  XOS resources.

* **Role:** A set of privileges granted to a User in the scope of
  some set of objects.

* **RootPrivilege:** Global scope. By virtue of having an account,
  every User implicitly has root privilege at one of two levels:

  - **Admin:** Read/write access to all objects.

  - **Default:** Read/write access to this User (with the exception of
    being able to change their home Site); read access to all objects
    associated with the User's home Site; and the right to list all
    Users, Sites, Deployments, and Nodes in the system.

* **SitePrivilege:** A binding of a User to a Role in the scope of a
  particular Site, which implies the Role applies to all Nodes,
  Slices, and Users associated with the Site. Site-level roles
  include:

  - **Admin:** Read/write access to all Site-specific objects.

  - **PI:** Read/write access to a Site's Users and Slices.

  - **Tech:** Read/write access to a Site's Nodes.

* **SlicePrivilege:** The binding of a User to a Role in the scope
  of a particular Slice, which implies the Role applies to all Slivers
  and Networks associated with the Slice. Slice-level roles include:

  - **Admin:** Read/write access to all Slice-specific objects.

  - **Access:** Allowed to access Slivers instantiated as part of the
    slice.

* **Deployment Privileges:** The binding of a User to a Role in the
  scope of a particular Deployment, which implies the Role applies
  to all Objects of type Image, NetworkTemplate, and Flavors
  assocaited with the Deployment. The sole Deployment-level role is:

  - **Admin:** Read/write access to all Deployment-specific objects.

Operationally, Root-level Admins create Sites, Deployments, and Users,
granting Admin privileges to select Users affiliated with Sites and
Deployments. By default, all Users are able to list the registered
Sites, Deployments, and Nodes; have read access to their home site;
and have read/write access to its own User Object.

Site Admins create Nodes, Users, and Slices associated with the Site. 
The Site Admin may also grant PI privileges to select Users (giving
them the ability to manage the Site's Users and Slices), and Tech
privileges to select Users (giving them the ability to manage the
Site's Nodes).

Site Admins and PIs grant Admin privileges to select Users affiliated
with the Site's Slices. These Users, in turn, add additional Users to
the Slice and instantiate the Slice's Sliver's and Networks. All Users
affiliated with a Slice may access (ssh into) the Slice's Slivers.

Deployment Admins manage Deployments, including defining their
operating parameters, specifying what Sites may contribute Nodes to
the Deployment, specifying what Sites may acquire resources from
the Deployment, and binding the Sites in the Deployment to one or
more Controllers.

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
network. These servers are aggregated along two dimensions, one
related to location and the other related to policy. This approach is
motivated by the need to decouple the hosting organization from the
operational organization, both of which collaborate to manage
nodes. This results in three core Object Types:

* **Node:** A physical server that can be virtualized.

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

* **Deployment:** A logical grouping of Nodes running a compatible set
  of virtualization technologies and being managed according to a
  coherent set of resource allocation policies.

  - Bound to a set of Users that establish the Deployment's policies.

  - Bound to a set of Nodes that adhere to the Deployment's policies.

  - Bound to a set of supported Images that can be booted on the
    Deployment's nodes.

  - Bound to a set of supported NetworkTemplates that can connect VMs
    in the deployment.

  - Bound to a set of supported Flavors that establish allocation
    parameters.

  - Bound to a set of Controllers that represent the back-end
    infrastructure service that provides cloud resources (e.g., an
    OpenStack head node).

Sites and Deployments can be one-to-one, which corresponds to a each
Site establishing its own policies. In practice, however, we expect
Deployments will often span multiple Sites, where those Sites either
correspond to a single distributed organization (e.g., Internet2) or
agree to manage their Nodes in collaboration with the containing
Deployment (e.g., Enterprise). It is also possible that a Site hosts
Nodes that belong to more than one Deployment.

Correspondingly, a Controller object represents the binding between
Deployments and Sites. There could be one Controller per Site in the
Deployment (i.e., one OpenStack head node per Site) or one Controller
might be bound to a set of Sites (i.e., one OpenStack head node
manages a set of Sites).

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

* **Controller:** An object that represents the binding of a site to
  a back-end service (e.g., an OpenStack head node). Includes the URL
  at which the back-end service controller can be invoked and the
  credentials needed to invoke it.

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

  - Bound to an Image that boots in each of the Slice's Slivers.

  - Optionally bound to a Service that defines the Slice's interface.

* **Sliver:** A single instance (VM) associated with a Slice. Each
  Sliver is instantiated on some physical Node.

* **Network:** A virtual network associated with a Slice. Each Network
 also specifies a (possibly empty) set of other Slices that may join
 it, has an IP AddrSpace (CIDR block and port range) by which all
 Slivers are addressed, is designated at Public or Private depending
 on whether its addresses are publicly routable, has a
 NetworkParameter that parameterizes the selected NetworkTemplate.

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

XOS maintains the invariant that all Slivers belonging to the Slice
are attached to all Networks associated with the Slice, with the
convention that every Network appears as an interface (in the sense of
an Unix interface, for example, eth1) within each Sliver.

There is intended asymmetry in the definition of a Slice. All Slivers
bound to a Slice share the same Image (hence that field is defined
Slice-wide), while each Network potentially has a different
NetworkTemplate (hence that field is defined per-Network).

###Services and Tenancy

XOS goes beyond Slices to define a model for the service running
within a Slice:

* **Service:** A registered network service that other Services can
  access via extensions to the XOS data model and API. Each Service is

  - Bound to a set of Slices that collectively implement the Service.

  - Bound to a set of Controller that repesents the service's control
    interface.

Operationally, service developers -- Users with Admin privileges for
the Slice(s) that implement a Service -- create Service objects, which
automatically registers them with XOS. Doing so may also involve
defining one or more Service-specific objects that extend the core
data model, and provide other users with a means to create an
"instance" of the Service for their use (e.g., create a "Volume" for
storage service). Extending XOS with these new Service-specific
objects is not currently a first-class operation in XOS, but rather,
involves directly augmenting the data model, as described in the
[Adding Services to XOS](../2_developer/#adding-services) section of the
Developer Guide.

The power of building a system from modular components is realized
when the system provides a means (framework) for composing those
components. XOS provides explicit support for composing services, or
to be more specific, for service providers to declare that ``Service A
is a tenant of Service B.'' We represent this relationship in the XOS
Data Model as:

* **Tenant:** A binding of a Tenant Service to a Provider Service,
  parameterized as a combination of:

  - **Connect:** Means by which the two services connect and exchange
    packets with each other. Options include: Public, Shared-Private,
    and Interconnect-Private.

  - **Attributes:** Tenancy related state required by provider service
    on behalf of a given tenant service.
