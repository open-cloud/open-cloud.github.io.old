---
layout: page
title: Roadmap
---

This section will eventually record a detailed roadmap. For now it is
a place to collect todo items, roughly divided into short-term fixes
and medium-term features (plus our deployment plans).

##Immediate Fixes and Upgrades

1. For slivers attached to a "Dedicated public IP" network, the IP
   address and SSH command shown on the Sliver Details page are both
   incorrect.

2. Automate the binding of interfaces to images.

3. Enhance and document Developer View's VN management interface.

4. Complete full range of Site Privilges.

5. Automatically manage multiple user keys pers slice.

6. SlicePrivilege=Default should mean the user may ssh into the
   slice's slivers (not "no special privilege"). Rename "Default" to
   "User".

7. Relation between ServiceClass and Flavor need attention.

8. Bring up Syndicate and integrate into Tenant view.

9. Prepare Docker image of "base" XOS and document installation.

10. Perform a complete security audit.

11. Enable monitoring mini-dashboard.

##Medium-Term Feature Development

1. Service Composition View/Language

2. OVX-based VN interconnection

3. Service Classes and Invoices

4. Bring up HPC; revisit user-visible API.

5. RequestRouter as a stand-alone service

6. User-contributed images

7. Enhanced monitoring and stats

8. Integrate OVX installation into the OpenStack install process

9. Docker support

10. Run ONOS in "domain0"

##Deployments

1. Migrate ViCCI servers to OpenCloud.

2. Bring up ViNI sites.

3. Autonomous OpenStack clusters (both private clouds and the HP
   public cloud).

4. Migrate OpenFlow-enabled clusters to OVX.

