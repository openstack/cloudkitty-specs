..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================================================
Deprecate `state` field and propose `last_processed_at` field
===============================================================

This spec proposes a name change for the `state` field in
`cloudkitty_storage_states` table. The goal is to use a more descriptive
and meaningful name. The proposed new column name is `last_processed_at`.



Problem Description
===================

The `state` column in `cloudkitty_storage_states` table holds the timestamp
of the last processing cycle for a scope. The problem with that name
is that it does not represent the data it stores. A column name like `state`
gives the expectation that we would have some info about the resource such as
`active`/`disabled`/`enabled` or something similar. However, that is not what
we get in the `state` column.


Proposed Change
===============

Therefore, to avoid confusions, and to adopt a more consistent naming for
this variable, we propose to rename the column to `last_processed_at`.
Moreover, we propose to deprecate the wording `state` in the scope API,
therefore, instead of using `state` to represent the last processing
timestamp, we would use `last_processed_at`.



Alternatives
------------

Other names can be considered. It is up for other people to propose,
and then we can discuss and try to reach a consensus.


Data model impact
-----------------

Rename the column `state` in the `cloudkitty_storage_states` table to
`last_processed_at`.


REST API impact
---------------

Introduce the new field `last_processed_at` alongside the `state`
field. Then, we announce its deprecation and remove it completely in after
Wallaby.

Security impact
---------------
None

Notifications Impact
--------------------
Renaming of the `state` field in the `cloudkitty_storage_states`, and as a
consequence in the scope API, we will deprecate it and introduce a new field
along side the `state` attribute.

Other end user impact
---------------------

None

Performance Impact
------------------

None

Other deployer impact
---------------------

None

Developer impact
----------------

None
Implementation
==============

Assignee(s)
-----------

Primary assignee:
 - Rafael <rafael@apache.org>


Work Items
----------

1) propose, discuss, and merge the spec
2) execute the implementations to rename the column, introduce the new
   attribute in the API, and then execute a warning against the use of the
   `state` variable.
3) implement changes in the CloudKitty client
4) After Wallaby, we remove the `state` field from the API

Dependencies
============
None

Testing
=======
Unit tested

Documentation Impact
====================
None

References
==========

None so far
