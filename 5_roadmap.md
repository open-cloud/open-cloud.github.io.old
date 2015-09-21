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
(e.g., CORD, OpenCloud) from a common release. Sub-goals include: (a)
use Docker to configure the development environment, and (b) use TOSCA
to specifying the service configuration.

2. Tenancy Model. The goal is to codify the tenancy model that came
out of the CORD proof-of-concept.

3. Reconcile with OpenStack. The goal is to eliminate gratuitous
differences between XOS and OpenStack, including (a) instances, and
(b) network ports.

4. Upgrade to OpenStack Kilo. This enables XOS to access cloud
resources in a domain of an existing OpenStack cluster.

5. New Observer/Synchronizer Framework. The goal is to transition to a
more robust and complete Observer (renamed Synchronizer). Sub-goals
include: (a) better integration of Ansible, and (b) support
inter-model policy plugins.

6. Monitoring Service. The goal is to incorporate OpenStack's
Ceilometer into XOS, making it available as a first-class XOS service
that other services (and views) and use. *[Not expected to be
available for September release of Burwell, but will be included in
subsequent deployment configurations (see below).]*

7. Integrate ONOS. The goal is to incorporate ONOS into XOS, replacing
the default OpenStack/Neutron networking support. *[Not expected to be
available for September release of Burwell, but will be included in
subsequent deployment configurations (see below).]*

##Deployment Configurations (Sept - Nov 2015)

XOS is a framework that is deployed on different hardware
configurations and populated with different service portfolios. We
expect the following deployment configurations to be based on the
Burwell release.

1. Development: A minimal configuration used for development, with backend
cloud resources provided by CloudLab. Corresponds to the environment
described in [Developer Guide](../2_developer). Available with the
Burwell release.

2. OpenCloud: Deployed across multiple Internet2 sites. Service
portfolio includes OpenStack, EC2, Syndicate, and vCDN. Available with
the Burwell release. (Extended to include Ceilometer in October and
ONOS in November.)

3. CORD Development: A configuration used for CORD development. 
Includes vCPE and Ceilometer, but no virtualized access devices. 
Leverages CloudLab servers rather than actual POD hardware. 
Avaiable in October 2015.

4. CORD POD: A configuration to be deployed on a CORD POD with OLT
hardware support. Service portfolio includes OpenStack, ONOS,
Ceilometer, vCDN, vOLT, vCPE, and vBNG. Avaiable in November 2015.
