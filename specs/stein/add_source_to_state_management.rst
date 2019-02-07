..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================
Add some details to state management
====================================

CloudKitty is becoming more and more generic, at all levels. With the current
state of the project, scopes can be anything, and can even be different when
several collectors are running with different configurations. In consequence,
some more details should be added to the storage state management.

Associated storyboard story: https://storyboard.openstack.org/#!/story/2004957

Problem Description
===================

Currently, CloudKitty only stores the ID of a scope in its SQL database when
checking/updating a scope's state. Given that scope IDs are UUIDs in most of the
cases (project IDs for example), this is (in theory) collision-safe. However,
since CloudKitty does now support multiple sources for data collection,
collisions are more likely to occur, especially if data about the same
resources is collected from different sources. In addition to that, a scope ID
is **not** required to be an UUID, even if it is considered best practice.

Example of a problematic configuration:

* Collector A collects data from monasca with the keystone fetcher and scope
  being ``project_id``.
* Collector B collects data from gnocchi with the keystone fetcher and scope
  being ``project_id``.

This does currently have unpredictable behavior, as both collectors would
update the same scope.


Proposed Change
===============

In order to improve support for multiple sources and avoid collisions or
unexpected behavior, the fetcher by which a scope ID was retrieved along with
the name of the collector that was used to retrieve the data should also be
stored in the state table. In order to avoid all hypothetic collisions, the
scope_key that was used for this specific scope ID should also be added.

Thus, each collection unit would be defined by the following tuple:
``(scope_id, scope_key, collector_name, fetcher_name)``.

This would allow collectors collecting data from different sources or with
different scopes to synchronize in a consistent way.

In order to be backward-compatible, the state manager will have the following
behavior: When checking a scope ``(A, B, C, D)``, it will try to retrieve the
state of the scope ``(A, B, C, D)``. If no such scope is found, it will try to
retrieve the scope ``(A, None, None, None)``. If that scope is found, its empty
columns will be updated with values ``B``, ``C``, and ``D``.

Alternatives
------------

One alternative would be to document discouraged configurations. However, this
would add some constraints and edge cases which could be supported with the
current proposal.

Data model impact
-----------------

Three nullable columns will be added to the ``cloudkitty_storage_states``
database, along to the ``identifier`` column:

* ``scope_key``
* ``collector``
* ``fetcher``

REST API impact
---------------

None

Security impact
---------------

None

Notifications Impact
----------------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

None

Other deployer impact
---------------------

None, the migration will be handled by the ``cloudkitty-dbsync upgrade``
command.

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  lukapeschke/peschk_l

Work Items
----------

* Add additional information to the state management table, and update the
  state checking logic.

Dependencies
============

None

Testing
=======

Given that this is just an algorithmic change (no new feature / API change /
behavior impact), unit tests will be enough. No tempest integration tests are
required.

Documentation Impact
====================

Some details about how the state of a scope is stored will be added to the
documentation.

References
==========

This has been discussed during the CloudKitty IRC meeting of February 1st 2019.
The logs of this meeting are available at http://eavesdrop.openstack.org/meetings/cloudkitty/2019/cloudkitty.2019-02-01-15.00.log.html
("Adding collector + fetcher to the state database" and "Q & A" topics,
starting at 15:42:26).
