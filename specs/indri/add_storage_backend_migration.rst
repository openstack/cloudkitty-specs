..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 creativecommons.org/licenses/by/3.0/legalcode

=====================================
Add storage backend migration support
=====================================

https://storyboard.openstack.org/#!/story/2011874

This specification proposes a command for offline migration of rated data
between CloudKitty v2 storage backends.


Problem Description
===================

CloudKitty has no supported way to move rated data when an operator changes
storage backend. Changing ``[storage] backend`` directs new data to the new
backend without copying previous data, leaving operators without historical
reports.

This is particularly relevant to deployments moving away from InfluxDB after
its removal from Kolla Ansible, but the problem is not specific to just them.
CloudKitty needs a method for operators to change backends whilst retaining the
data frames.


Proposed Change
===============

A new ``cloudkitty-storage-migrate`` command will copy rated data between two
v2 storage backends. The backend configured by ``[storage] backend`` will be
the destination. The source will be selected with ``--migrate-from``. Existing
backend-specific configuration will provide connection details for both
backends

For example, the following configuration and command will migrate InfluxDB data
to OpenSearch::

  [storage]
  version = 2
  backend = opensearch

  [storage_influxdb]
  # InfluxDB connection options

  [storage_opensearch]
  # OpenSearch connection options

  $ cloudkitty-storage-migrate --migrate-from influxdb

The first implementation will have the following scope:

* Only certain v2 storage backends will be supported.

* Migration will be offline. All services writing rated data must be stopped.

* The source and destination must be different backends.

* The destination must be provisioned and initialized with
  ``cloudkitty-storage-init`` before migration.

* Dataframes will be copied through the storage driver ``retrieve`` and
  ``push`` methods.

* Copying will be paginated. ``--page-size`` will default to 1000 records.

* The range will use half-open ``[begin, end)`` semantics. Optional ``--begin``
  and ``--end`` values may restrict it.

* A new ``get_bounds`` driver method will return the earliest dataframe start
  and latest end. An empty source will return ``{'begin': None, 'end': None}``
  and result in a no-op.

* A non-empty destination range will be rejected by default.
  ``--overwrite-destination`` will explicitly permit deletion through the
  destination driver's ``delete`` method, limited to the migration range.

* Verification will compare record counts, overall quantity and price totals,
  and totals grouped by metric type. A persistent mismatch will exit non-zero.

The initial implementation will support InfluxDB, Elasticsearch, and
OpenSearch. Loki will not be supported initially because it does data deletion
asynchronously and will require more complex handling.

Only rated data will be copied. SQL-backed storage and reprocessing state and
rating configuration will not change.

Alternatives
------------

Database native tools or deployment project specific migration workflows.

Implementing migration in a deployment project such as Kolla Ansible would
serve only that project's workflow. Native database tools would require
converting data for every backend pair.


Data model impact
-----------------

There will be no change to the format of CloudKitty dataframes or to any SQL
database schema.

The v2 storage interface will gain a ``get_bounds`` method. Its base
implementation will raise ``NotImplementedError`` so existing third-party
drivers remain usable outside migration.


REST API impact
---------------

None.


Security impact
---------------

The command will use existing storage credentials.

``--overwrite-destination`` allows for destructive deletion within the selected
range and will be disabled by default.


Notifications Impact
--------------------

None.


Other end user impact
---------------------

Users will not interact with this feature through the REST API. A new command,
``cloudkitty-storage-migrate``, will be added to the CLI.


Performance Impact
------------------

None, with the exception of the expected performance impact of different
backends, which is outside the scope of this specification. The command is
desgined to be run once, offline, and will not be part of the normal CloudKitty
service operation.


Other deployer impact
---------------------

The command needs valid connection details for both drivers. The destination
must be provisioned and configured as the active backend.

A typical migration would look like this:

#. Stop every service that writes rated data, normally all
   ``cloudkitty-processor`` instances.
#. Provision the destination and configure both the source and destination
   driver connection details.
#. Set ``[storage] backend`` to the destination and run
   ``cloudkitty-storage-init``.
#. Run ``cloudkitty-storage-migrate --migrate-from <source>``.
#. Check that the command completed successfully
#. Start the processor services again.

Developer impact
----------------

No new library is required. The v2 storage interface will gain a ``get_bounds``
method. Its base implementation will raise ``NotImplementedError`` so existing
third-party drivers remain usable. New drivers need to implement ``get_bounds``
to support migration.


Implementation
==============


Assignee(s)
-----------

Primary assignee:
  dawudm

Work Items
----------

* Implement ``get_bounds`` for supported drivers.
* Add the migration command and needed helpers.
* Add unit and fake-backend migration tests.
* Add operator documentation and a release note.
* Document unsupported backends.


Dependencies
============

None.


Testing
=======

Unit tests will cover bounds, validation, pagination, overwrite, failures, and
verification. Fake storage clients will simulate full migrations and compare
records and totals across supported backend pairs.

Documentation Impact
====================

* A new "Storage backend migration" page will be added to the
  "Command-Line Interface Reference". It will describe how to configure and
  initialize the source and destination backends and use the migration
  command, including time bounds, pagination, destination overwrite, and
  unsupported backends.

References
==========

* CloudKitty implementation proposal: https://review.opendev.org/c/openstack/cloudkitty/+/1000891

* Kolla Ansible removal of InfluxDB and Telegraf deployment support:
  https://review.opendev.org/c/openstack/kolla-ansible/+/973050
