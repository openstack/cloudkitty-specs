..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================================================
V1 API rating modules endpoints migration to V2 API
===================================================

https://storyboard.openstack.org/#!/story/2006572

With the v1 API being frozen, migrating endpoints that are not present in the v2
API is required to achieve a complete v2 API with both existing v1 features and
new v2 ones. This spec is about the ``/v1/rating/modules`` endpoints.

Problem Description
===================

As the v1 API is planned to be deprecated, ``/v1/rating/modules`` endpoints must
be ported to v2. The motivation for the v2 API is documented in the relevant
spec_.

.. _spec: https://specs.openstack.org/openstack/cloudkitty-specs/specs/stein/adding_a_v2_api_to_cloudkitty.html


Proposed Change
===============

Implementing the existing ``/v1/rating/modules`` into the v2 API. This is part of
the broader effort to migrate v1 endpoints to the v2 API.

Alternatives
------------

None.

Data model impact
-----------------

None.

REST API impact
---------------

This will add ``/v2/rating/modules`` endpoints similarly to the v1 API.

``GET /v2/rating/modules`` returns the list of loaded modules. It does not require
any parameter. The body of the response will contain the module list:

.. code-block:: javascript

   {
     "modules": [
       {
           "module_id": "f6f4f726-583z-4a09-a972-c908dea4c291"
           "description": "Sample extension.",
           "enabled": true,
           "hot-config": false,
           "priority": 2
       },
       [...]
     ]
   }


``GET /v2/rating/modules/<module_id>`` returns the module specified by the id in
the url. It does not need any parameter. The body of the response contains a
single module:

.. code-block:: javascript

   {
      "module_id": "f6f4f726-583z-4a09-a972-c908dea4c291",
      "description": "Sample extension.",
      "enabled": true,
      "hot-config": false,
      "priority": 2
   }


``PUT /v2/rating/modules/<module_id>`` changes the state and priority of a module.
It requires fields to be updated to be present in the query body:

.. code-block:: javascript

   {
     "enabled": false,
     "priority": 42
   }

For both GET endpoints, the expected response code is a 200 OK.
For the PUT endpoint, the expected response is a 204 NO CONTENT as the body is
empty.
Expected HTTP error response codes for all 3 endpoints are:

* ``400 Bad Request``: Malformed request.

* ``401 Unauthorized``: The user is not authenticated.

* ``403 Forbidden``: The user is not authorized.



Security impact
---------------

All added endpoints will keep the same policy as the v1 API: admin credentials
are needed to query the API endpoints. Added endpoints only port existing
functionality from v1 to v2 API and will not have any other security impact.

Notifications Impact
--------------------

A notification reload triggered by queries on added endpoints is planned in
the future.

Other end user impact
---------------------

None.


Performance Impact
------------------

None.

Other deployer impact
---------------------

None.

Developer impact
----------------

None.

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

* Implement the ``/v1/rating/modules`` endpoints in the v2 API, including
  unit tests and documentation

* Add functional tests to the tempest plugin

* Add support to CloudKitty's client

Dependencies
============

None.

Testing
=======

Besides regular unit tests, the tempest plugin will be updated to add new tests
for added endpoints.


Documentation Impact
====================

v2 API reference will be updated to reflect changes.

References
==========

None.
