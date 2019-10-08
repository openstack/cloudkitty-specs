..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================================
Cloudkitty monasca fetcher implementation
=========================================

https://storyboard.openstack.org/#!/story/2006675

Monasca is already supported in cloudkitty through its collector. However,
cloudkitty lacks a dedicated fetcher for monasca, relying instead on other
fetchers.


Problem Description
===================

No scope discovery is available when using the monasca collector. This forces
users to use the keystone fetcher and assign the 'rating' role to every project
where cloudkitty is needed. Another less than ideal solution is to use the
'source' fetcher, but that requires to specify every scope in the configuration
file.


Proposed Change
===============

Implementing a new fetcher using monasca to discover scopes:

- The fetcher will connect to monasca and retrieve a list of scopes (dimensions
  values in monasca) given a specified dimension name. This is achieved through
  monasca's python client and its ``metrics.list_dimension_values()`` method.
- The monasca endpoint will be retrieved from keystone in similar fashion to the
  monasca collector, as it is needed for the monasca client initialization.
  This is a good opportunity to mutualize the monasca's client bootstraping code
  in a common file.

Configuration options include:

- ``dimension_name``: the monasca dimension from which the scope_ids should be
  retrieved, defaults to ``project_id``.
- ``monasca_tenant_id``: The monasca tenant id, has no default value.
- ``monasca_service_name``: Name of the monasca service, defaults to ``monasca``
- ``interface``: Endpoint type, defaults to ``internal``

Alternatives
------------

None

Data model impact
-----------------

None

REST API impact
---------------

None

Security impact
---------------

The fetcher will use the same authentication mechanism as the current
monasca collector (keystone for authentication and python-monascaclient for
monasca interactions), and won't introduce any new dependency.
Thus, the new fetcher shouldn't have any security impact.

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
  qanglade/qanglade
Other contributors:
  lukapeschke/peschk_l

Work Items
----------

* Implement a cloudkitty monasca scope fetcher using python-monascaclient.

Dependencies
============

No new dependency is needed.

Testing
=======

Regular unit tests will be included.


Documentation Impact
====================

Documentation of the new fetcher will be added, mostly covering configuration
of the fetcher.


References
==========

None.
