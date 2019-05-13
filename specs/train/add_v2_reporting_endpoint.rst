..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================
Add a v2 reporting endpoint
===========================

https://storyboard.openstack.org/#!/story/2005664

Problem Description
===================

Now that the v2 storage has been merged, new features can be implemented on
top of it. This is a proposal for a v2 endpoint replacing v1's
``/report/summary``. The endpoint should support filters and grouping on any
``groupby`` attribute.

Proposed Change
===============

The proposed endpoint will support pagination, a dict of filters, and a list of
attributes to group data by.

Example usecases:

* A per-user total for the current month, for the whole cloud:

  Example client usage::

    cloudkitty summary get --groupby user_id

  This would result in the following HTTP request::

    GET /v2/summary?groupby=user_id

* A per-resource total for scope X and user Y for a specific week:

  Example client usage::

    cloudkitty summary get --filter project_id:X --filter user_id:Y --groupby id --begin 2019-05-01T00:00:00 --end 2019-05-07T00:00:00

  This would result in the following HTTP request::

    GET /v2/summary?filter=project_id:X&filter=user_id:Y&groupby=id&begin=2019-05-01T00%3A00%3A00&end=2019-05-07T00%3A00%3A00

Alternatives
------------

None

Data model impact
-----------------

There is no direct change to the data model. However, the v2 storage interface
will have to be updated: no distinction should be made between ``groupby`` and
``metadata`` filters. Thus, the ``group_filters`` parameter should be removed
from the prototypes of the ``retrieve`` and ``total`` function.

REST API impact
---------------

This will add an endpoint on ``/v2/summary`` with support for the ``GET``
HTTP method.

The method will support the following parameters:

* ``offset``: (optional, defaults to 0) The offset of the first element that
  should be returned.

* ``limit``: (optional, defaults to 100) The maximal number of results to
  return.

* ``filter``: (optional) A dict of metadata filters to apply.

* ``groupby``: (optional) A list of attributes to group by.

* ``begin``: (optional, defaults to the first day of the month at midnight)
  start of the period the request should apply to.

* ``end``: (optional, defaults to the first day of the next month at midnight)
  end of the period the request should apply to.

A ``GET`` request on this endpoint will return a JSON object of the following
format:

.. code-block:: yaml

   type: "object"
   properties:
     total:
       type: "integer"
     columns:
       type: "array"
       items:
         type: "string"
     results:
       type: "array"
       items:
         oneOf:
          - type: string
          - type: string
            format: datetime
          - type: integer
          - type: float

``total`` contains the total number of matching elements (for pagination).

``columns`` contains the list of columns available for each element of
``results``.

``results`` is a list of same-length arrays sorted in the same way as
``columns``.

Example response:

.. code-block::

   GET /v2/summary?groupby=user_id

   {
     "total": 20,
     "columns": [
        "begin",
        "end",
        "qty",
        "rate",
        "user_id"
     ],
     "results": [
        [
          "2019-05-01T00:00:00Z",
          "2019-06-01T00:00:00Z",
          42.21,
          13.37,
          "f6b331ad-af19-45b9-a4a3-2d27e8ab76e0"
        ],
        [...]
     ]
   }

Security impact
---------------

There is no security impact introduced by this patch.

.. note:: In order to limit access, requests from non-admin users will
          automatically have a filter on their project added.

Notifications Impact
--------------------

None

Other end user impact
---------------------

The client's ``summary get`` method will be updated when using the v2 API in
order to provide support for pagination, filters and grouping. User experience
will be improved, as users will have a more precise view of their resource
usage.

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
  peschk_l

Other contributors:
  jferrieu

Work Items
----------

* Update the prototype of the ``retrieve`` and ``total`` methods of the v2
  storage interface in order to remove the ``group_filters`` parameter.

* Implement the endpoint.

* Add tempest tests for this endpoint.

* Add support for the endpoint to the client.

Dependencies
============

None

Testing
=======

Tempest tests for this endpoint will be added.

Documentation Impact
====================

The endpoint will be added to the API reference.

References
==========

None
