..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================================================
Add a v2 API endpoint to retrieve DataFrame objects
===================================================

https://storyboard.openstack.org/#!/story/2005890

CloudKitty needs an endpoint in its v2 API to retrieve DataFrame objects.
This spec aims at defining what is to be done and why. Everything in
this document is open to discussion, equally on the associated
storyboard and gerrit.

Problem Description
===================

Now that there is an endpoint available bound on ``POST /v2/dataframes`` that
allows us to push DataFrames objects into the CloudKitty storage backend,
we also need a v2 API endpoint to retrieve these objects from it.

Also this feature is available in the v1 API on ``GET /v1/storage/dataframes``.
Therefore it is necessary to have it in the v2 API.

Proposed Change
===============

The proposed endpoint will support pagination, a dict of filters,
a ``begin`` and an ``end`` parameter as ISO 8601 strings that support
timezone offsetting e.g. 2019-08-29T07:18:40+00:00.

It will be available on ``GET /v2/dataframes``.

Example:

* Getting all dataframes for ``image.size`` metric type for project X and user Y
  for a specific week:

  This would result in the following HTTP request::

    GET /v2/dataframes?filter=type:image.size&filter=project_id:X&filter=user_id:Y&begin=2019-05-01T00:00:00&end=2019-05-07T00:00:00

The feature will also be available to the CloudKitty client library and CLI.

Alternatives
------------

None.

Data model impact
-----------------

None.

REST API impact
---------------

This will add an endpoint on ``/v2/dataframes`` with support for the ``GET``
HTTP method.

The endpoint will support the following query parameters:

* ``offset``: (optional, defaults to 0) The offset of the first element that
  should be returned.

* ``limit``: (optional, defaults to 100) The maximal number of results to
  return.

* ``filter``: (optional) A dict of metadata filters to apply.

* ``begin``: (optional, defaults to the first day of the month at midnight)
  start of the period the request should apply to.

* ``end``: (optional, defaults to the first day of the next month at midnight)
  end of the period the request should apply to.


A ``GET`` request on this endpoint will return a JSON object of the following
format:

.. code-block:: javascript

   {
       "total": 2,
       "dataframes": [
           {
               "usage": {
                   "volume.size": [
                       {
                           "vol": {
                               "unit": "GiB",
                               "qty": 1.9
                           },
                           "rating": {
                               "price": 3.8
                           },
                           "groupby": {
                               "project_id": "8ace6f139a1742548e09f1e446bc9737",
                               "user_id": "b28fd3f448c34c17bf70e32886900eed",
                               "id": "be966c6d-78a0-42cf-bab9-e833ed996dee"
                           },
                           "metadata": {
                               "volume_type": ""
                           }
                       }
                   ]
               },
               "period": {
                   "begin": "2019-08-01T01:00:00+00:00",
                   "end": "2019-08-01T02:00:00+00:00"
               }
           },
           {
               "usage": {
                   "image.size": [
                       {
                           "vol": {
                               "unit": "MiB",
                               "qty": 3.55339050293
                           },
                           "rating": {
                               "price": 1.77669525146
                           },
                           "groupby": {
                               "project_id": "5994682e63af4aa8873d247aa28b876e",
                               "user_id": "b28fd3f448c34c17bf70e32886900eed",
                               "id": "f7541950-96d5-4e89-ac76-9d4eacf59b83"
                           },
                           "metadata": {
                               "container_format": "foo",
                               "disk_format": "bar"
                           }
                       }
                   ]
               },
               "period": {
                   "begin": "2019-08-01T02:00:00+00:00",
                   "end": "2019-08-01T03:00:00+00:00"
               }
           }
       ]
   }


The expected HTTP success response code for a ``GET`` request on this endpoint
is ``200 OK``.

Expected HTTP error response codes for a ``GET`` request on this endpoint are:

* ``400 Bad Request``: Malformed request.

* ``403 Forbidden``: The user does not have the necessary rights
  to retrieve dataframes.

* ``404 Not Found``: No dataframes were found for the provided parameters.

This endpoint will be authorized for admins and for the tenant/project owners
of the requested dataframes (regarding the specified ``project_id`` filter).

Security impact
---------------

Any user with access to this endpoint will be able to retrieve information about
data rated by CloudKitty. Thus, access to this endpoint should be granted
to non-admin users with parsimony.

Notifications Impact
--------------------

None.

Other end user impact
---------------------

The client will also be updated in order to include a function and a CLI command
allowing to retrieve DataFrame objects.

Performance Impact
------------------

None.

Other deployer impact
---------------------

Dataframes will be easier to retrieve for admins and deployers.

Developer impact
----------------

None.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  <jferrieu>

Work Items
----------

* Implement the API endpoint with unit tests

* Add tempest tests

* Support this endpoint in the client with unit and functional tests.

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

* Spec: Add DataFrame/DataPoint objects:
  https://specs.openstack.org/openstack/cloudkitty-specs/specs/train/add_dataframe_datapoint_object.html

* Spec: Add a v2 API endpoint to push DataFrame objects:
  https://specs.openstack.org/openstack/cloudkitty-specs/specs/train/add_push_dataframes_api_endpoint.html
