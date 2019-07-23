..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================================
Add a v2 API endpoint to push DataFrame objects
===============================================

https://storyboard.openstack.org/#!/story/2005890

CloudKitty needs an endpoint in its v2 API to push DataFrame objects.
This spec aims at defining what is to be done and why. Everything in
this document is open to discussion, equally on the associated
storyboard and gerrit.

Problem Description
===================

With the current state of CloudKitty, the only way to import
DataFrames into the CloudKitty storage is to load a storage driver and to call
the ``push`` method but it is impossible through the CloudKitty API.

This inconvenience prevents CloudKitty from having external processors being able to
easily push data into its storage. e.g. CloudKitty cannot have DataFrame fixtures
provisioned so it can be functionally tested with OpenStack Tempest.

A new admin API endpoint is needed to allow this.

The Tempest tests would also enable a more efficient workflow for validating forthcoming
features and therefore improve the overall quality and stability of them in a long term
goal and, more generally speaking, having this feature implemented would also represent
an opportunity to anyone to extend their usage of CloudKitty in a more handy and flexible
way.

Proposed Change
===============

A new endpoint will be made available to admin users on ``POST /v2/dataframes``.
This will allow end users to push DataFrames in the form of JSON objects.

Alternatives
------------

We could keep the old way, but external processors would need to be written in Python
to be able to load the storage driver modules plus they would need the storage backend
credentials exposed to them.

Otherwise CloudKitty could be provisioned by using more hackish ways
such as manipulating directly the storage backend and the database metadata but
this is a laborious and error-prone process that should definitely be avoided.


Data model impact
-----------------

None.

REST API impact
---------------

This will add an endpoint on ``/v2/dataframes`` with support for the ``POST``
HTTP method.

The endpoint will support the following body parameters:

* ``dataframes``: (required)
  A json array describing DataFrame objects (see below for details).

Inside the body of the request, a collection of DataFrame json objects
can be specified as follows:

.. code-block:: javascript

  {
    "dataframes": [
      # first DataFrame
      {
        "period": {
          "begin": "20190723T122810Z",  # valid ISO 8601 datetime format
          "end": "20190723T132810Z"     # valid ISO 8601 datetime format
        },
        "usage": {
          "metric_one": [  # list of DataPoint json objects
            {
              "vol": {
                "unit": "GiB",
                "qty": 1.2
              },
              "rating": {
                "price": 0.04
              },
              "groupby": {
                "group_one": "one",
                "group_two": "two"
              },
              "metadata": {
                "attr_one": "one",
                "attr_two": "two"
              }
            }
            # { second DataPoint for metric_one here }
          ],
          "metric_two": []
        }
      }
      # { second DataFrame here }
    ]
  }

The expected HTTP success response code for a ``POST`` on this endpoint
is ``204 No Content``. This decision have been made in order to reduce
the network bandwidth consumption for this kind of operation.
The user will be able to consult DataFrames through a ``GET /v2/dataframes``
endpoint that will be available in the future.

Expected HTTP error response codes for a ``POST`` request on this endpoint are:

* ``400 Bad Request``: Malformed request.

* ``401 Unauthorized``: The user is not authenticated.

* ``403 Forbidden``: The user is not authorized to push DataFrame objects.

This endpoint will only be authorized for administrator end users.

Security impact
---------------

Any user with access to this endpoint will be able to alterate data on
the target platform which is an action carrying heavy side effects.
Thus, access to this endpoint should be granted to non admin users with
uttermost concern or not at all if possible.

Notifications Impact
--------------------

None.

Other end user impact
---------------------

The client will also be updated to include a function and a CLI command
allowing to push DataFrame objects.

Performance Impact
------------------

None.

Other deployer impact
---------------------

Importing DataFrame objects of any nature to CloudKitty will now be
an easy process. This will be handy to provision CloudKitty if felt
necessary.

Developer impact
----------------

Importing DataFrame objects will allow to push fixtures into
the CloudKitty storage and therefore to add scenarios to the Tempest plugin.
This will be handy to write integration tests afterwards.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  <jferrieu>

Other contributors:
  <peschk_l>

Work Items
----------

* Implement the API endpoint with unit tests.

* Add tempest tests.

* Support this endpoint in the client.

Dependencies
============

This endpoint depends on the DataFrame and DataPoint objects specification:

* Spec: https://review.opendev.org/#/c/668669/

Testing
=======

Unit tests and Tempest tests for this endpoint will be added.

Documentation Impact
====================

The endpoint will be added to the API reference.

References
==========

Spec: Add a v2 API endpoint to push dataframe objects:

https://review.opendev.org/#/c/668669/
