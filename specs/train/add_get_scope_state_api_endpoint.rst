..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================================================
Add a v2 API endpoint to retrieve the state of different scopes
===============================================================

https://storyboard.openstack.org/#!/story/2005395

.. note:: This spec is the detailled implementation of one part of a
          larger spec. If you haven't read it yet, please see
          https://review.opendev.org/#/c/657393/

Problem Description
===================

For now, there is no way for admins to know until which date a scope's data has
been processed, except by looking at the ``cloudkitty_storage_states`` SQL
table. This is, however, critical information to have before an invoice/report
is generated, as you have to make sure that the whole period you're billing
for has been processed.

Proposed Change
===============

This information should be made available through the API. This describes a
V2 API endpoint. It will only be available to admins.

Alternatives
------------

None.

Data model impact
-----------------

None.

REST API impact
---------------

This will add an endpoint on ``/v2/scope`` with support for the ``GET``
HTTP method.

A ``GET`` request on this endpoint returns a paginated list of all scopes,
with support for optional filters on the scope, the fetcher, the collector
and/or the scope_key.

The method will support the following parameters:

* ``offset``: (optional, defaults to 0) The offset of the first scope that
  should be returned.

* ``limit``: (optional, defaults to 100) The maximal number or results to
  return.

* ``scope_id``: (optional) Can be specified several times. One or several
  scope_ids to filter on.

* ``collector``: (optional) Can be specified several times. One or several
  collectors to filter on.

* ``fetcher``: (optional) Can be specified several times. One or several
  fetchers to filter on.

* ``scope_key``: (optional) Can be specified several times. One or several
  scope_keys to filter on.

The response will have the following format:

.. code-block:: javascript

   {
       "results": [
           {
               "collector": "gnocchi",
               "fetcher": "keystone",
               "scope_id": "7a7e5183264644a7a79530eb56e59941",
               "scope_key": "project_id",
               "state": "2019-05-09 10:00:00"
           },
           {
               "collector": "gnocchi",
               "fetcher": "keystone",
               "scope_id": "9084fadcbd46481788e0ad7405dcbf12",
               "scope_key": "project_id",
               "state": "2019-05-08 03:00:00"
           },
           {
               "collector": "gnocchi",
               "fetcher": "keystone",
               "scope_id": "1f41d183fca5490ebda5c63fbaca026a",
               "scope_key": "project_id",
               "state": "2019-05-06 22:00:00"
           }
       ]
   }

The expected HTTP success response code for a ``GET`` request on this endpoint
is ``200 OK``.

Expected HTTP error response codes for a ``GET`` request on this endpoint are:

* ``400 Bad Request``: Malformed request.

* ``403 Forbidden``: The user hasn't the necessary rights to retrieve the state of
  the scopes.

* ``404 Not Found``: No scope was found for the provided filters.

This endpoint will only be authorized for admins.

Security impact
---------------

Any user with access to this endpoint will be able to retrieve information about
the state of all scopes rated by cloudkitty. Thus, access to this endpoint should
be granted to non-admin users with parsimony.

Notifications Impact
--------------------

None.

Other end user impact
---------------------

The client will also be updated in order to include a function and a CLI command
allowing to retrieve information about the state of the different scopes.

Performance Impact
------------------

None.

Other deployer impact
---------------------

Scope state information will be easier to retrieve for admins and deployers.

Developer impact
----------------

None.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  <peschk_l>

Work Items
----------

* Implement the API endpoint with unit tests

* Add tempest tests

* Support this endpoint in the client.

Dependencies
============

None.

Testing
=======

Tempest tests for this endpoint will be added.

Documentation Impact
====================

The endpoint will be added to the API reference.

References
==========

Spec to get/reset the state of a scope: https://review.opendev.org/#/c/657393/
