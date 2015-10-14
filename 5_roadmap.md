---
layout: page
title: Roadmap
---

This section records high-level the plan of record, but is typically
less detailed than information on the
[Development Mailing List](https://groups.google.com/a/xosproject.org/forum/#!forum/devel).
Also see bug reports and short-term feature requests filed at the   [GitHub Issue
Tracker](https://github.com/open-cloud/xos/issues).

##Cozad Release (December 2015)

1. Separate subsystems into their own containers. For example,
    XOS and Postgress should run in separate containers. Synchronizers
    should also be isolated in their own containers.  We also need to do
    some design work around how to best extend the data model schema.

2. Leverage OpenStack domains so XOS can run on autonomous
    OpenStack clusters.

3. Extend support for "subscribers" (and other "service users") by
    taking advantage of external identity management systems; e.g.,
    via OpenID. 

4. Related to (3), revisit the XOS data model for users, roles, and
    privileges.

5. Improve UI programming environment, including (a) re-engineering
    xoslib to support Single Page Application, and (b) re-factoring
    to support a Js StyleGuide.

6. Provide a complete set of views, including
   * Operator: Complete "XOS Admin", Nagios, Horizon, ELK-Stack.
   * Developer: Evolve "Tenant View" into "Developer View".
   * Service User: Content Provider (CDN), Syndicate, CORD Subscriber.
   * *[Note: Current "Developer View" conflates Operator, Developer, 
     and Subscriber perspectives. Re-factor as outlined above.]*

7. Integrate Monitoring Service into XOS. The plan is to make
    OpenStack's Ceilometer a first-class XOS service, including
   instrumentation of all software services.

8. Integrate ONOS into XOS. The plan is to make ONOS a
   first-class XOS service, including replacing the default
   OpenStack/Neutron networking support with ONOS.

9. Define a data model migration strategy (and stick with it). The
    current strategy too often depends on resetting the data model.

##Deployment Configurations (Oct - Dec 2015)

XOS can be deployed on different hardware configurations and populated
with different service portfolios. We expect the following deployment
configurations to be based on the Burwell release.

1. Development: A minimal configuration used for development, with backend 
cloud resources provided by CloudLab. Corresponds to the environment 
described in [Developer Guide](../2_developer). Available with the 
Burwell release.

2. Test: A configuration used used to run regression tests on the XOS
core, with backend cloud resources provided by CloudLab. Available
with the Burwell release.

3. OpenCloud: Deployed across multiple Internet2 sites. Service
portfolio includes OpenStack, EC2, Syndicate, and vCDN. Available with
the Burwell release. (Extended to include Ceilometer in October and 
ONOS in November.)

4. CORD Development: A configuration used for CORD development. 
Includes vCPE, ONOS, and minimal vOLT and vBNG. Leverages CloudLab
servers rather than actual POD hardware. Avaiable in October 2015.
(Extended to include Ceilometer in October and ONOS in November.)

5. CORD POD: A configuration to be deployed on a CORD POD with OLT
hardware support. Service portfolio includes OpenStack, ONOS,
Ceilometer, vCDN, vOLT, vCPE, and vBNG. Avaiable in November 2015.
