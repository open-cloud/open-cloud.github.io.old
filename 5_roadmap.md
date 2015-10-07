---
layout: page
title: Roadmap
---

This section records high-level the plan of record, but is typically
less detailed than information on the
[Development Mailing List](https://groups.google.com/a/xosproject.org/forum/#!forum/devel).
Also see bug reports and short-term feature requests filed at the [GitHub Issue
Tracker](https://github.com/open-cloud/xos/issues).

##Cozad Release (December 2015)

1. Improved support for isolating extensions and Sychronizers in their
   own containers.

2. Able to leverage OpenStack domains so XOS can run on autonomous
   OpenStack clusters.

3. Extend support for "subscribers" by taking advantage of external
   identity management systems, for example using OpenID.

4. Monitoring Service fully integrated into XOS. The plan is to make
   OpenStack's Ceilometer a first-class XOS service, including
   instrumentation of all software services.

5. ONOS fully integrated into XOS. The plan is to make ONOS a
   first-class XOS service, including replacing the default
   OpenStack/Neutron networking support with ONOS. 

##Deployment Configurations (Oct - Dec 2015)

XOS can be deployed on different hardware configurations and populated
with different service portfolios. We expect the following deployment
configurations to be based on the Burwell release.

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
