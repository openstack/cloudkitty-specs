..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================================================
Introduce reprocessing API
===============================================================

This specification proposes an API to enable reprocessing.


Problem Description
===================

Sometimes due to a misconfiguration in rating rules, or due to
backfill of data in the collector backend (Gnocchi, Prometheus,
or others), operators need to reprocess CloudKitty data. The
reprocessing process that is normally taken is the following.

- Stop all CloudKitty processors,
- reset the `state` (last processed/rated timestamp) in MySQL database for all
  scopes that need reprocessing,
- clean/delete all processed/rated data for the affected timeframe in the
  processed data backend (e.g. InfluxDB),
- restart CloudKitty processors.

CloudKitty has an API already to reset the processing/rating
state of a scope [1]_. However, this API only resets the state
in the MySQL database. Therefore, it is important to have
an API that implements the manual steps that operators are
normally executing when reprocessing is required.


Proposed Change
===============

To avoid human error during reprocessing, and also to make
it easier for non-CloudKitty experts to execute it, we propose
the introduction of a reprocessing API that schedules the
reprocessing and also removes the processed/rated data for the
affected timeframe from the backend (e.g. InfluxDB).


To achieve that, we will use a new table to store the reprocessing
progress for scopes, the proposed table name is ``storage_scope_reprocessing_schedule``.

.. code-block:: SQL

  +------------------------------+--------------+------+-----+-------------------+----------------+
  | Field                        | Type         | Null | Key | Default           | Extra          |
  +------------------------------+--------------+------+-----+-------------------+----------------+
  | id                           | int(11)      | NO   | PRI | NULL              | auto_increment |
  | identifier                   | varchar(256) | NO   | MUL | NULL              | FK             |
  | start_reprocess_time         | datetime     | NO   |     | -                 |                |
  | end_reprocess_time           | datetime     | NO   |     | -                 |                |
  | current_reprocess_time       | datetime     | yes  |     | NULL              |                |
  | reason                       | text         | NO   |     | NULL              |                |
  +------------------------------+--------------+------+-----+-------------------+----------------+

The endpoint for the schedule reprocessing API will be:

- ``POST`` ``/v2/task/reprocesses``

  - ``scope_id``: a list of scope IDs to schedule reprocessing to.
  - ``start_reprocess_time``: a timestamp to start reprocessing.
  - ``end_reprocess_time``: a timestamp to end reprocessing.
  - ``reason``: the reason for the reprocessing

- ``GET`` ``/v2/task/reprocesses/<scope_id>`` -- to retrieve the
  reprocessing schedules for a single scope.

- ``GET`` ``/v2/task/reprocesses`` - to retrieve the
  reprocessing schedules for multiple scopes


The ``/v2/task/reprocesses`` endpoint will receive a POST request similar to
the following:

.. code-block:: python

    {
      "scope_id": ["scope_id_one", "scope_id_two", ...],
      "start_reprocess_time": "2021-05-01 00:00:00",
      "end_reprocess_time": "2021-05-30 23:00:00"
      "reason": "The reason why this reprocessing is scheduled."
    }

The reason field is mandatory to force users to explain why the
reprocessing is scheduled. We are forcing users to register why
they are taking such drastic measures, such as reprocessing.
Therefore, we hold a history of the scheduled reprocessing, and
their respective reasons/explanations.

If one of the scope IDs informed via ``scope_id`` does not exist,
we will raise an exception telling the operator that there are
invalid scopes ID in the request.

One can only schedule reprocessing for timestamps of scopes that
have already been processed. Therefore, if the `end_reprocess_time`
is after the latest processing timestamp for a given resource, an
error is thrown in the API.

One cannot schedule overlapping reprocessing. Therefore, it is
only possible to reprocess a reprocessing of a scope, when the
previously scheduled reprocessing is finished (if there is an
overlapping of start/end timestamps).

After the schedule has been created, the CloudKitty processors
will clean the affected time range for the given resource, and
then move on with the reprocessing. Every time a timestamp is
reprocessed, the ``current_reprocess_time`` is updated, similarly
to the current processing workflow.

When the ``current_reprocess_time`` equals to ``end_reprocess_time``,
it means that the scheduled reprocessing has finished. Therefore,
The reprocessing workers will do nothing for this entry.

We will also introduce an endpoint to consult the status of the
scheduled reprocessing.

- To list all schedules: ``GET`` ``/v2/task/reprocesses``

  - ``scope_id``: optional field to filter scopes that
    have been scheduled for reprocessing. If no scope
    is provided, we list all.


Alternatives
------------

Manually execute the reprocessing steps previously described.


Data model impact
-----------------

A new table (``storage_scope_reprocessing_schedule``) will be
added to the database.


REST API impact
---------------

A new API will be introduced: ``/v2/task/reprocesses``.

Security impact
---------------

Only admins must have access to this new API

Notifications Impact
--------------------
None

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
2) execute the implementations as described.
3) implement changes in the CloudKitty client to support the new API

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

.. [1] https://docs.openstack.org/cloudkitty/latest/api-reference/v2/index.html?expanded=get-a-rating-summary-detail,reset-the-status-of-several-scopes-detail#reset-the-status-of-several-scopes
