---
layout: page
title: Release Notes
---

##To Fix Before Release

1. Remove visible references to Reservations, Accounts, Billing, and
   Invoices. [DONE]

2. Need to either hide or document the ability to whitelist Networks.
   [DONE: limitation documented below]

3. Hide undocumented Views. Keep only Tenant, Developer, Nagios, and 
   xsh. [DONE]

4. Complete Site Privilges (Admin, Tech, PI, Default) or document as
   incomplete. [DONE: limitation documented below]

5. Nodes tab should be visible from the Sites page. [DONE]

6. Tags tab should not be visible from the Sites page (only from
   Slices). [DONE]

7. Finish the example Acceess Control policy language example in the
   Administering a Deployment section. [DONE]

8. Why does the Deployment page include both an Images selector and
   an Images tab. Seems like the latter is unnecessary. If that's not
   the case, document the tab. [DONE]

9. Make sure user's can't set/unset Admin and Read-Only buttons in
   Login Details without the proper authorization. (And do we really
   need Read-Only any more?)

10. Users->Dashboard View != the set of views visible on the user's
   home page (and does not reflect changes made through the Customize
   tab).

11. Slice objects should not be saved if creator is not set. A
    non-admin should not be able to set the creator to someone else.

12. Determine if Sliver.IP is a useful field or redundant with
    NetworkSlivers. Update admin to display the appropriate one (right
    now we display comma-separated NetworkSlivers addresses in Sliver
    list, but Sliver.IP in Sliver detail view)

13. In the Developer view, a status of "1-OK" still shows up as a red
    error symbol.

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




