---
layout: page
title: Roadmap
---

This section records high-level the plan of record, but is typically
less detailed than information on the
[Development Mailing List](https://groups.google.com/a/xosproject.org/forum/#!forum/devel).
Also see bug reports and short-term feature requests filed at the [GitHub Issue
Tracker](https://github.com/open-cloud/xos/issues).

##Burwell Release (September 2015)

1. Configuration Management. The goal is to remove deployment
dependencies from the code base so we can support multiple deployments
(e.g., CORD, OpenCloud) from a common release. Also use Docker to
simplify the development environment and TOSCA as a means of
specifying the deployment configuration.

2. Tenancy Model. The goal is to codify the tenancy model that came
out of the CORD proof-of-concept.

3. Reconcile with OpenStack. The goal is to eliminate gratuitous
differences between XOS and OpenStack (e.g., Instances instead of Slivers).

4. Upgrade to OpenStack Kilo. This enables XOS accessing cloud
resources in a domain of an existing OpenStack cluster.

5. New Observer/Synchronizer Framework. The goal is to transition to
a more robust and complete Observer,  which we will likely rename as
the Synchronizer. Sub-goals include: (a) better integration of
Ansible, and (b) support inter-model plugins (e.g., HPC and OpenStack).

6. Instrumentation-as-a-Service. The goal is to incorporate
OpenStack's Ceilometer into XOS, making it available as a first-class
XOS service that other services (and views) and use. *[Not expected to
be available for September release of Burwell, but will be included in
subsequent deployment configurations (see below).]*

7. ONOS. The goal is to incorporate ONOS into XOS, replacing the
default OpenStack/Neutron networking support. *[Not expected to
be available for September release of Burwell, but will be included in
subsequent deployment configurations (see below).]*

##Deployment Configurations (Sept - Nov 2015)

XOS is a framework that is deployed on different hardware
configurations and populated with different service portfolios. We
expect the following deployment configurations to be based on the
Burwell release.

1. Development. Minimal configuration used for development, with
backend cloud resources provided by CloudLab. Corresponds to the
environment described in [Developer Guide](../2_developer). Available
with the Burwell release.

2. OpenCloud. Deployed across multiple Internet2 sites. Service
portfolio includes OpenStack, EC2, ONOS, Ceilometer, Syndicate, and
vCDN. Available in October 2015.

3. CORD POD: Deployed on CORD PODs with OLT hardware support. Service
portfolio includes OpenStack, ONOS, Ceilometer, vCDN, vOLT, vSG
(formerly vCPE), and vRouter (formerly vBNG). Avaiable in November
2015.
