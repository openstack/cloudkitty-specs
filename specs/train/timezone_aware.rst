..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================
Making CloudKitty timezone-aware
================================

Associated story: https://storyboard.openstack.org/#!/story/2005319

Currently, CloudKitty is not timezone-aware and every timestamp is considered
to be UTC. In order to improve user experience, avoid confusion and potential
bugs (when doing time object conversions), CloudKitty should be made
timezone-aware.

Problem Description
===================

Internally, CloudKitty manipulates timezone-unaware ``datetime`` objects and
timestamps. Every object representing some point in time is considered to be
UTC. This can be confusing for users (which expect the timestamps of their data
to be in their current timezone) as well as a source of bugs: A ``datetime``
object (converted to an iso8601 format timestamp) has no timezone information.
There is no guarantee about how this kind of string will be interpreted by
another API: as UTC, or in the API's local time ?

Proposed Change
===============

In order to secure CloudKitty's behaviour about timezones, the following points
should be applied:

* All ``datetime`` objects must be made timezone-aware.

* CloudKitty must only manipulate ``datetime`` objects. Functions or interfaces
  accepting several types of objects for parameters containing time information
  must be changed in order to only accept timezone-aware ``datetime`` objects.

* The conversion to/from iso8601 timestamps or unix timestamps must be done at
  the interface level. This means that conversions to/from timezone-aware
  ``datetime`` objects must only be done by CloudKitty's HTTP API and the
  different drivers (storage, fetcher, collector, messaging...), and only when
  this type is not supported.

* Any timezone-unaware object retrieved by the API must be considered as UTC,
  made timezone-aware and raise a warning.

* The handling of timezone-aware objects is not consistent between different
  SQL databases. In consequence, the storage state will still be stored with
  no timezone information and will be stored as UTC. The storage state driver
  will be in charge of making objects aware/unaware and will only
  provide/accept timezone-aware ``datetime`` objects. To avoid complex
  migrations and inconsistencies, the same rule will apply to the timestamps of
  dataframes: They will be stored as timezone-unaware timestamps and considered
  to be UTC. The storage driver will be responsible for the conversions.

* Client functions querying the v2 API will send iso8601 timestamps with
  timezone information. If no timezone information is provided in the CLI
  arguments, they will be considered as local time and adequate timezone
  information will be added.

For convenience, microseconds will not be supported for now, and will be set
to 0 in all ``datetime`` objects.

Alternatives
------------

- Leaving the code as is and making clear in the documentation that everything
  is UTC. This does not prevent unexpected behaviour when sending
  timezone-unaware info to other APIs.

Data model impact
-----------------

None, the storage state model is not modified.

REST API impact
---------------

None.

Security impact
---------------

None.

Notifications Impact
--------------------

None.

Other end user impact
---------------------

For v2 API endpoints, timestamps with no timezone information passed to the
client will be considered as localtime by the client, and timezone information
will be updated accordingly.

Performance Impact
------------------

None.

Other deployer impact
---------------------

None. The upgrade will impact neither the state storage nor the dataframes
storage as no storage migration will be required. In consequence, the existing
data will not be impacted.

Developer impact
----------------

Clear and stricter rules will reduce potential time-related bugs in new
features implementation.

Implementation
==============

Assignee(s)
-----------

Gerrit topic:
  timezone-aware

Primary assignee:
  peschk_l

Other contributors:
  None

Work Items
----------

Some of the following points are highly interdependent and may need to be
implemented in the same commit:

- Removing usage of Unix timestamps from the codebase in order to only
  manipulate ``datetime`` objects internally.

- Adding timezone information to all ``datetime`` objects manipulated
  internally.

- Make the functions of the client querying the v2 API timezone-aware.

Dependencies
============

- ``python-dateutil``: Library providing extensions to the standard library's
  ``datetime`` module.

Testing
=======

Unit tests are sufficient until the v2 API has more endpoints. Tempest tests
testing v2 API endpoints should be ran with and without timezone information
in timestamps to ensure that the API has the expected behaviour.

Documentation Impact
====================

The v2 API documentation and client documentation will include a description
of their behaviour with timezone-aware/unaware timestamps.

References
==========

None.
