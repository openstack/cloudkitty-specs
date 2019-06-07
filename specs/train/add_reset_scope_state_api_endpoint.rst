..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================================================
Add a v2 API endpoint to reset the state of different scopes
============================================================

https://storyboard.openstack.org/#!/story/2005395

CloudKitty needs an endpoint int its v2 API to reset the state of one or
several scopes. This spec aims at defining what is to be done and why.
Everything in this document is open to discussion, equally on the associated
storyboard and gerrit.

Problem Description
===================

With the current state of CloudKitty, if an administrator is willing to reset
calculations for a given scope for any reason e.g. changes in CloudKitty's
collection configuration, update for some rating rules, etc. He has to stop all
the processors, update the state of said scope in the database and restart all
the processors. This is a laborious process and we should make an easier way
available to the end user in the form of a new v2 API endpoint.

Proposed Change
===============

A new endpoint will be made available to admin users on ``PUT /v2/scope``
with relatively similar parameters that are to be found on the
``GET /v2/scope`` endpoint regarding filtering. This will allow
end users to reset the scope state of several scopes at once if they are
willing to. There is an additional parameter detailed later for preventing the
end user from performing any irreversible change too quickly.

Alternatives
------------

We could only allow resetting scope state on ``PUT /v2/scope/<scope_id>`` but
that would prevent the end user to reset several scope states at once and
would enforce one request per scope state reset attempt.

Also, a list of the effected scopes by the request could be returned to the end
user but we have already a specified endpoint ``GET /v2/scope`` for this use
case. That wouldn't be therefore necessary and would add dispensable complexity
to the implementation for no expected usage.

Data model impact
-----------------

None.

REST API impact
---------------

This will add an endpoint on ``/v2/scope`` with support for the ``PUT`` HTTP
method.

The endpoint will support the following parameters:

* ``all_scopes``: (mutually exclusive with ``scope_id``, defaults to False)
  Whether we target all scopes at once or not. This parameter intends to
  prevent the end user from doing any mistake too quickly. If all_scopes and
  scope_id parameters are both absent or present no data would be affected by
  the request and a ``400 Bad Request`` response will be returned.

* ``scope_id``: (mutually exclusive with ``all_scopes``) Can be specified
  several times. One or several scope_ids to filter on to reset associated
  scope states. If all_scopes and scope_id parameters are both absent or
  present no data would be affected by the request and a ``400 Bad Request``
  response will be returned.

* ``collector``: (optional) Can be specified several times. One or several
  collectors to filter on to reset associated scope states.

* ``fetcher``: (optional) Can be specified several times. One or several
  fetchers to filter on to reset associated scope states.

* ``scope_key``: (optional) Can be specified several times. One or several
  scope_keys to filter on to reset associated scope states.

As the scope state reset is a time consuming operation and relies on the
underlying AMQP system, it will be an asynchronous call to the API and will
respond to the caller with a ``202 Accepted``, once the notifications have been
sent across the message queue. The response body of this request is empty.

The expected HTTP success response code for a ``PUT`` request on this endpoint
is therefore ``202 Accepted``.

Expected HTTP error response codes for a ``PUT`` request on this endpoint are:

* ``400 Bad Request``: Malformed request. This will also happen in the case
  where ``all_scopes`` and ``scope_id`` parameters are both present or missing.

* ``403 Forbidden``: The user is not authorized to reset the state of any scope.

* ``404 Not Found``: No scope was found for the provided parameters.

This endpoint will be only authorized to admins.

Security impact
---------------

Any user with access to this endpoint will be able to perform a scope state
reset on one or several scopes which is an action carrying heavy side effects.
Thus, access to this endpoint should be granted to non-admin users with
uttermost concern or not at all if possible.

Notifications Impact
--------------------

Notifications will be sent to an AMQP listener in order to trigger the scope
state reset. The structure of the notification to be sent will be the following:

.. code-block:: javascript

    {
       "scopes": [
           {
               "collector": "gnocchi",
               "fetcher": "keystone",
               "scope_id": "7a7e5183264644a7a79530eb56e59941",
               "scope_key": "project_id"
           },
           {
               "collector": "gnocchi",
               "fetcher": "keystone",
               "scope_id": "9084fadcbd46481788e0ad7405dcbf12",
               "scope_key": "project_id"
           },
           {
               "collector": "gnocchi",
               "fetcher": "keystone",
               "scope_id": "1f41d183fca5490ebda5c63fbaca026a",
               "scope_key": "project_id"
           }
        ]
    }


Other end user impact
---------------------

The client will also be updated in order to include a function and a CLI command
allowing to reset state for one or several given scopes.

Performance Impact
------------------

The deletion operation carried by the scope state reset call will depend on the
amount of data associated with the targeted scopes and may be time consuming,
impairing the database performance in the same course.

Other deployer impact
---------------------

Scope state reset will cease to be a tricky and laborious process and will
profit admins and deployers for operating scopes.

Developer impact
----------------

None.


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

Spec to get/reset the state of a scope: https://specs.openstack.org/openstack/cloudkitty-specs/specs/train/reset_scope_state.html
