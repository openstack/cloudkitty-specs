..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================
Adding a v2 API to CloudKitty
=============================

https://storyboard.openstack.org/#!/story/2004208

CloudKitty has gone through a lot of changes recently (v2 storage, support for
Prometheus, standalone mode...). This requires big changes on the API: given
that the internal data format is evolving, we need to expose the data
differently, in order to support the new features of the v2 storage: pagination,
grouping, filtering...

Problem Description
===================

Several distinct problems can be identified:

1. CloudKitty's data needs to be exposed with more granularity. A non-exhaustive
   list of currently missing features:

     * Pagination
     * Filtering
     * Grouping
     * ...

2. For its current API, CloudKitty uses WSME + pecan. Code written using these
   tools can be quite difficult to understand for new contributors. In addition
   to that, pecan's development seems to have slowed down:
   https://github.com/pecan/pecan/commits/master .

3. Semantics. A lot of endpoints are still using OpenStack-related terms
   (tenant, service...).

Proposed Change
===============

In consequence to the previously evoked reasons, the CloudKitty development
team proposes to implement a new API, which will be based on Flask/Werkzeug
and Flask-RESTful. These frameworks have been chosen for the following reasons:

- Flask is a well-known and easy-to-use framework. This will make contributions
  easier for new contributors. It is solid and used by other OpenStack projects,
  like Keystone.

- Flask-RESTful makes code easier to read and routing is also eased. The idea
  would be to have each important endpoint (for example ``/rating/hashmap``)
  mapped to a blueprint, and each sub-endpoint (``rating/hashmap/services``)
  mapped to a Flask-RESTful resource. This would make adding new endpoints easy,
  and leaves space for the v2 API to evolve. Below a working code sample, with
  voluptuous input and output validation:

  .. code-block:: python

     @api_utils.add_output_schema(GET_OUTPUT_EXAMPLE_SCHEMA)
     def get(self):
         policy.authorize(flask.request.context, 'example:get_example', {})
         return {}

     @api_utils.add_input_schema(POST_INPUT_EXAMPLE_SCHEMA)
     def post(self, fruit='banana'):
         policy.authorize(flask.request.context, 'example:submit_fruit', {})
         if fruit not in ['banana', 'strawberry']:
             raise http_exceptions.Forbidden(
                 'You submitted a forbidden fruit',
             )
         return {
             'msg': 'Your fruit is a ' + fruit
         }

- Flask-RESTful forces (or at least encourages) contributors to have a restful
  approach to the API, whereas Flask leaves them too much freedom.

This new API will make it easy to add a new endpoint, and will be a container
to new features. Any new endpoint will be added to this API.

This v2 API will only be compatible with the v2 storage, allowing new endpoints
to completely exploit the capacities of the v2 storage, without having to handle
backward compatibility. The v1 API is completely compatible with the v2 storage,
so data will still be accessible through the v1 API, compatibilty with
existing scripts etc, will be kept. The v2 API will be disabled when a v1
storage backend is used.

In order to limit the workload, not all v1 endpoints will be migrated at once.
The proposition will be to have one WSGI app per API version, and to
distinguish between them through routing. This could be done with Werkzeug's
``DispatcherMiddleware``: http://werkzeug.pocoo.org/docs/0.14/middlewares/#werkzeug.wsgi.DispatcherMiddleware.
In case a v1 storage backend is used, the v2 API will simply not be loaded.

V1 endpoints will be migrated one after another. Once the entire v1 API has
been migrated, the v2 API will be marked as ``CURRENT``, and the v1 API will
be marked as ``DEPRECATED``. Until that point is reached, v1 will be
``CURRENT`` and v2 will be ``EXPERIMENTAL``.

The proposed workflow for adding an endpoint is the following:

- Write a spec detailing what the endpoint does, its impact on the client,
  the dashboard and the tempest plugin.

- **Once the spec is reviewed and merged**, create a storyboard story
  linking to your spec and containing one task for each of the following points:

  * Adding the endpoint to the API

  * Adding support for the endpoint to the client

  * Adding tests for this endpoint to the tempest plugin

  * (Optional) Adding support for the endpoint to the dashboard

  Each of these tasks must be the subject of a new patch. Each task which is
  not marked as optional is mandatory.

- The patches adding client/dashboard support as well as the patch adding
  tests to the tempest plugin must depend on the patch adding the endpoint
  to the API.

Once the v2 API is stable and marked as the current API, it will be
**microversioned** in order to have an indicator of its capabilities. A
microversion increase will be indicating a new feature (new endpoint).
However, API endpoints in OpenStack's service catalog will **not** be
microversioned. The exact version of the API will be available at the API root
(``/``).

Versioning will be done using **semantic versioning** (https://semver.org/).
Once all v1 endpoint have been migrated, and the v2 API is considered stable,
its version will be ``2.0``. Before that version, each version will be suffixed
with ``-beta.version``.

Example: ``v2.0-beta.1`` -> ... -> ``v2.0-beta.x`` -> ``v2.0``.

Alternatives
------------

Migrate the whole v1 API to Flask and extend existing endpoints in order to
support the features listed above: This would imply to migrate the whole API
codebase at once, which can't be done. Also, this wouldn't address the
semantics problem.

Data model impact
-----------------

This particular change (creating a v2 WSGI app and changing the way requests
are routed) has no impact on the datamodel. Each v2 API endpoint will be the
subject of a new spec. Potential data model impact will be detailed in these
upcoming specs.

REST API impact
---------------

* v1 API will go from ``EXPERIMENTAL`` to ``CURRENT`` current state.

* A v2 API, marked as ``EXPERIMENTAL``, will be available.

* A new route (``/v2``) will be available. It will contain a placeholder
  indicating that this will be the prefix for all upcoming v2 endpoints.

Security impact
---------------

None

Notifications Impact
--------------------

None

Other end user impact
---------------------

None. However, ``python-cloudkittyclient`` will need to be compatible with the
new v2 endpoints.

Performance Impact
------------------

Pagination will allow to lower the load on the network.

Other deployer impact
---------------------

None

Developer impact
----------------

For developers, adding a new enpoint should be way easier.

A dependency on Flask and Flask-RESTful will be added.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  lukapeschke/peschk_l

Gerrit topic:
  cloudkitty-v2-api

Work Items
----------

* Mark the v1 API as ``CURRENT``

* Route the API with Werkzeug instead of Pecan

* Add an Example endpoint showing how to implement an endpoint, and how to
  document it. This example endpoint should be removed as soon as the first
  real endpoint is implemented.

* Update the documentation: Update the developer documentation with an
  explanation about how to add an enpoint and generate the documentation of
  the v2 API.

Dependencies
============

* Flask

* Flask-RESTful

* os-api-ref (for API reference generation)

Testing
=======

This particular feature will be tested with unit tests only (gabbi and
unittest). Endpoints to come will also add some tests to the tempest plugin.

Documentation Impact
====================

The developer documentation will be updated to detail how requests are
routed and how to add a new endpoint. The user documentation will also be
updated in order to include the v2 API reference. The API reference will be
generated with ``os-api-ref``.

References
==========

* os-api-ref documentation: https://docs.openstack.org/os-api-ref/latest/

* API WG wiki:  https://wiki.openstack.org/wiki/API_Special_Interest_Group
