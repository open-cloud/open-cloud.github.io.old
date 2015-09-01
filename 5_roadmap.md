---
layout: page
title: Roadmap
---

This section records high-level the plan of record, but is typically
less detailed than information on the
[Development Mailing List](https://groups.google.com/a/xosproject.org/forum/#!forum/devel).
Also see bug reports and short-term feature requests filed at the [GitHub Issue
Tracker](https://github.com/open-cloud/xos/issues).

##Priorities for Burwell Release (September 2015)

1. Configuration Management. The goal is to remove deployment
dependencies from the code base so we can support multiple deployments
(e.g., CORD, OpenCloud) from a common git branch. Also use Docker to
simplify the development environment.

2. Tenancy Model. The goal is to codify the tenancy model that came
out of the CORD proof-of-concept.

3. Reconcile with OpenStack. The goal is to eliminate gratuitous
differences between XOS and OpenStack (e.g., Instances instead of Slivers).

4. Upgrade to OpenStack Kilo.

5. New Observer/Synchronizer Framework. The goal is to transition to
a more robust and complete Observer,  which we will likely rename as
the Synchronizer. Sub-goals include: (a) better integration of
Ansible, and (b) support inter-model plugins (e.g., HPC and OpenStack).

6. Instrumentation-as-a-Service. The goal is to incorporate
OpenStack's Ceilometer into XOS, making it available as a first-class
XOS service that other services (and views) and use.

##Future Releases

1. Relation between ServiceClass and Flavor need attention.

2. Support OVX-based VN interconnection

3. Re-establish Service Classes and Invoices

4. Make RequestRouter as a stand-alone service

5. Support user-contributed images

6. Enhance monitoring and stats

7. Integrate Docker support

8. Run ONOS in "domain0"

##Deployments

1. Migrate additional ViCCI servers to OpenCloud.

2. Bring up ViNI sites.

3. Transition OpenFlow-capable clusters to OVX.

