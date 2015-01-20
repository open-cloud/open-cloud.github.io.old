---
layout: page
title: Release Notes
---

##To Fix Before Release

1. Remove visible references to Reservations, Accounts, and
   Invoices.

2. Need to either hide or document the ability to whitelist Networks.

3. Hide undocumented Views. Keep only Tenant, Developer, and xsh.

4. Complete Site Privilges (Admin, Tech, PI, Default) or document as
   incomplete.

5. Nodes tab should be visible from the Sites page.

6. Tags tab should not be visible from the Sites page (only from
   Slices).

7. Finish the example Acceess Control policy language example in the
   Administering a Deployment section.

8. Why does the Deployment page include both an Images selector and
   an Images tab. Seems like the latter is unnecessary. If that's not
   the case, document the tab.

9. Make sure user's can't set/unset Admin and Read-Only buttons in
   Login Details without the proper authorization. (And do we really
   need Read-Only any more?)

10. Users->Dashboard View != the set of views visible on the user's
   home page (and does not reflect changes made through the Customize
   tab).

##Incomplete and Undocumented Features

1. There is only partial support for ServiceClass object.
   BestEffort is the only available class, and related objects
   (e.g., Reservations, Accounts, Invoices) are incomplete. 
   The relationship between the BestEffort class and Flavors
   needs attention.

2. Two Networks are automatically created and assigned to every
   Slice. There is sufficient support to create new networks and bind
   them to a slice, but the workflow is brittle and not documented.

3. A current limitation is that only one user key is injected into the
   slice. That user can login and manually add the keys of other users.

4. Need to automate the binding of interfaces to images.