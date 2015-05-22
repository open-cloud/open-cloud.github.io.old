---
layout: page
title: Release Notes
---

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



