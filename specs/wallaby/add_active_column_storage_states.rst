..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================================
Add "active" column to storage scope, and API to manage it
==========================================================
This spec proposes an extension for the "scope state" endpoint. The proposal
goal is to add a new option called "active" in the storage scope table
(cloudkitty_storage_states), and an API that enables operators to manage it.

Problem Description
===================

When using Gnocchi as a fetcher (so, we can process resources that are not
just OpenStack ones), we noticed that CloudKitty keeps processing resources
that have already been deleted in the systems they were created. Therefore,
they do not have measures in Gnocchi anymore. This causes a slowdown in the
processing of CloudKitty.

To present some figures, in a production environment, there are only ~330
resources that we want to process, but in CloudKitty storage states table, we
can see ~990 (this number just grows with time), and CloudKitty is processing
all of them.


During our meeting on January 11
(http://eavesdrop.openstack.org/meetings/cloudkitty/2021/cloudkitty.2021-01-11-14.00.html),
we discussed that maybe we could do that automatically. However, when I
reviewed the code changes that I needed, this would not be that simple. A
project in OpenStack could, for instance, have a single VM, and then this VM
could be deleted, and the project would not have measures anymore. Therefore,
we would mark it as inactive. Then, if the project receives VMs in the future
they would not be billed. Therefore, we would need to introduce a method to
change resources from inactive to active states, which is not that simple, and
therefore, this other mechanism would need to somehow process inactive
resources at some point in time to check if they still do not have measures in
Gnocchi.


Therefore, we came up with a simpler solution.

Proposed Change
===============

The proposal is to add two new fields in the storage state table
(cloudkitty_storage_states). A boolean column called "active", which indicates
if the CloudKitty scope is active for billing, and another one called
"scope_activation_toggle_date" (timestamp field) would store the latest
timestamp when the scope moved between the active/deactivated states.

Then, during CloudKitty processing, we would check the "active" column. If the
resource is not active, we could ignore it during the processing.

Moreover, we propose an API to allow operators to set the "active" field. The
"scope_activation_toggle_date" will not be exposed for operators to change it.
It would be updated automatically according to the changes in the "active"
field.

The proposal will add a new HTTP method to "/v2/scope" endpoint. We then will
use "patch" HTTP method to allow operators to patch a storage scope. The API
will require the scope_id, and then, it takes into account some of the fields
we allow operators to change, and "active" field is one of them.


Assignee(s)
-----------

Primary assignee:
  Rafael Weingärtner <rafael@apache.org>

Work Items
----------

1) Implement proposed changes
2) Update documentation and samples
3) implement changes in CloudKitty and OpenStack python clients

Dependencies
============
None
