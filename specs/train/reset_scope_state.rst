..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Allowing to get/reset the state of a scope
==========================================

https://storyboard.openstack.org/#!/story/2005395

Problem Description
===================

With the current state of CloudKitty, if an admin wants to reset calculations
for a given scope for any reason (changes in cloudkitty's collection
configuration, update of some rating rules...), he has to stop all the
processors, update the state of said scope in the database, and restart all
the processors. Furthermore, there is no way for an admin to get the current
state of a scope, unless he directly queries the SQL database.

Proposed Change
===============

.. warning:: This spec only covers the global schema. If it is accepted, three
             related specs detailing each part of the solution will be
             proposed.

In order to solve this, the following three changes are proposed:

* Add a ``delete`` method to the v2 storage interface allowing to delete all
  dataframes after a specific timestamp for one or several scopes. This will be
  the subject of another spec.

* Add a v2 API endpoint allowing to reset the state of one or several scopes.
  This will be the subject of another spec.

* Add a v2 API endpoint allowing to retrieve the state of one or several
  scopes. This endpoint will be a replacement for v1's ``/report/tenants``.
  This will be the subject of another spec.

The proposed logic is the following:

* The admin sets the status of a scope through the API. He can regularly check
  if the status of the previously updated scope has been updated through the
  scope state endpoint.

* The API endpoint sends an AMQP notification indicating that one or several
  scopes should be reset to a given state. Once an AMQP reply has been received
  by the API, either a 202 Accepted or 400 Bad Request status code will be
  returned (it is implied that authentication-related issues leading to 401 or
  403 errors will have been handled before reaching this point).

* Upon receiving a notification, the associated AMQP listener goes through the
  following steps:

  - The notification is parsed and validated. An AMQP reply indicating if it is
    valid or not is returned to the API. If the notification was invalid, the
    listener stops here.

  - For each scope, the following steps are applied:

    * To prevent a scope from being processed while its state is being reset
      and its data being deleted, the listener acquires a tooz lock. This
      operation is blocking.

    * Once the lock is acquired, all the data after the given state is
      deleted from the storage backend.

    * After the data has been deleted, the state of the scope is set to the
      given state.

    * Once this is done, the lock is released.

Alternatives
------------

This could be done without the AMQP notification. However, this would imply to
execute the locking and data update/deletion from the API, which would lead to
request timeouts.

Data model impact
-----------------

None.

REST API impact
---------------

The REST API impact will be detailed in the specs related to the API endpoints
described above.

Security impact
---------------

This will allow admins to delete user data through the API. This must be used
with care.

Notifications Impact
--------------------

This will add a notification and a new AMQP listener to the processor. The exact
change will be described in the associated spec.

Other end user impact
---------------------

None.

Performance Impact
------------------

Depending on the amount of data associated to a scope, the deletion may take
a consequent amount of time and affect the performance of the database.

Moreover, the scope to reset will be locked during data deletion, and processors
will thus not be able to process it.

Other deployer impact
---------------------

This will help admins, as resetting a scope won't require to manipulate the
database directly anymore.

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
   peschk_l

Other contributors:
   jferrieu

Work Items
----------

* Write a spec to add a ``delete`` method to the v2 storage interface allowing
  to delete all dataframes after a specific timestamp for one or several scopes.

* Write a spec to add a v2 API endpoint allowing to reset the state of one or
  several scopes.

* Write a spec to add a v2 API endpoint allowing to retrieve the state of one
  or several scopes. This endpoint will be a replacement for v1's
  ``/report/tenants``.

Dependencies
============

None.

Testing
=======

Testing will be detailed in each of the associated specs.

Documentation Impact
====================

* The API reference will be updated

* Some documentation and examples on how to reset the state of a scope will be
  added.

References
==========

None
