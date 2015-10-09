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

1. Support isolating Sychronizers in their own containers.
   We need to do some design work around how to best extend 
   the data model schema.

2. Leverage OpenStack domains so XOS can run on autonomous
    OpenStack clusters.

3. Extend support for "subscribers" (and other "service users") by
   taking advantage of external identity management systems; e.g.,
   via OpenID. 

4. Related to (3), revisit the XOS data model for users, roles, 
   and privileges.

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
