---
layout: page
title: Release Notes
---

##To Fix Before Release

1. Slice objects should not be saved if creator is not set. [DONE] A
    non-admin should not be able to set the creator to someone else.

2. Determine if Sliver.IP is a useful field or redundant with
    NetworkSlivers. Update admin to display the appropriate one (right
    now we display comma-separated NetworkSlivers addresses in Sliver
    list, but Sliver.IP in Sliver detail view)

3. Normal users should not be able to change their site.

4. Generate RESTful API documentation.

5. Have "Documentation" link at top of all pages point to Guide rather 
    than ../admin/doc.

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




