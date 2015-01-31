---
layout: page
title: Release Notes
---

##To Fix Before Release

1. A non-admin should not be able to set the creator of a slice to 
   someone else. [Tony]

2. Normal users should not be able to change their site. [Tony]

3. Documentation link at top of all pages point to Guide rather 
    than admin/doc. [Done]

4. Hide monitoring mini-dashboard for now. [Done]

5. A non-admin should not be able to write Deployment state. [Tony]

6. Augment Ansible to tie OpenCloud passwords to OpenStack passwords. [Sapan]

7. Delete non-sync'ed objects. [Sapan]

8. Documentation
   * REST API [Scott]  
   * Adding Services to XOS [Sapan]  

9. Attempting to use the REST API while not logged in returns "AttributeError: 'AnonymousUser' object has no attribute 'is_readonly'" [SCOTT]

10. REST API: /plstackapi/ should be renamed to /xos/ [SCOTT]

##Incomplete and Undocumented Features

1. There is only partial support for ServiceClass object.
   BestEffort is the only available class, and related objects
   (e.g., Reservations, Accounts, Invoices) are incomplete. 

2. Two Networks are automatically created and assigned to every
   Slice. There is sufficient mechanism to create new networks, bind
   them to a slice, and whitelist other slices that are allowed to
   connect to a network, but the workflow is brittle and not well
   documented. In particular, OpenStack limitations require that a
   Network be created before the VMs that are to attach to them, and
   adding a Network to a Slice requires re-instantiating all the
   associated Slivers.

3. A current limitation is that only one user key is injected into the
   slice. That user can login and manually add the keys of other users,
   but an OpenCloud admin also needs to add the keys to the account used
   to support proxy login to the slice.

4. The full range of SitePrivileges is not implemented. Only Admin 
   (currently named "PI") is supported, as opposed to Admin, Tech,
   and PI.

5. The services (Syndicate, HyperCache, RequestRouter) are not
   currently running. They will be instantiated once the release
   is stable.

## Installation Notes

1. some files need to be manually copied when installing using WSGI
    sudo mkdir /var/www/planetstack/static/rest_framework_swagger
    sudo cp -a /usr/local/lib/python2.7/dist-packages/rest_framework_swagger/static/rest_framework_swagger/* /var/www/planetstack/static/rest_framework_swagger/


