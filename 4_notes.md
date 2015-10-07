---
layout: page
title: Release Notes
---

###Burwell / 1-Oct-2015.

1. Improved configuration management, as documented in [Configuration
   Management](../2_developer/#config-mgmt) section of the Developer's
   Guide. Includes removing deployment dependencies from the code base
   to better support multiple deployments (e.g., CORD, OpenCloud) from
   a common release; the use Docker to configure the development
   environment; and the use TOSCA to specifying the service
   configuration.

2. Codified the tenancy model, as documented in the [Services and
   Tenancy](../0_overview.md/#service-tenancy) section of the
   Architecture Guide.

3. Reconciled with OpenStack by eliminating gratuitous differences,
   instances and network ports.

4. Upgraded to OpenStack Kilo. Support for Kilo's domain feature
   is not yet available.

5. Replace Synchronizer framework, as described in
   the [Adding Services to XOS](../2_developer/#adding-services)
   section of the Developer's Guide. Support is included for
   inter-Synchronizer coordination, but with limitations.

###Axtell / 1-May-2015.

1. Two Networks are automatically created and assigned to every
   Slice. There is sufficient mechanism to create new networks, bind
   them to a slice, and whitelist other slices that are allowed to
   connect to a network, but the workflow is brittle and not well
   documented. In particular, OpenStack requires that a
   Network be created before the VMs that are to attach to them, and
   adding a Network to a Slice requires re-instantiating all the
   associated Slivers.

2. A current limitation is that only one user key is injected into the
   slice. That user can login and manually add the keys of other users,
   but an OpenCloud admin also needs to add the keys to the account used
   to support proxy login to the slice.

3. Virtual networks are implemented as GRE-tunnels using the default
   Neutron plugin. OVX is currently running only on a development
   cluster.

4. There is only partial support for ServiceClass object.
   BestEffort is the only available class, and related objects
   (e.g., Reservations, Accounts, Invoices) are currently disabled.

5. Some files need to be manually copied when installing using WSGI

```
sudo mkdir /var/www/xos/static/rest_framework_swagger

sudo cp -a /usr/local/lib/python2.7/dist-packages/rest_framework_swagger/static/rest_framework_swagger/* /var/www/xos/static/rest_framework_swagger/
```



