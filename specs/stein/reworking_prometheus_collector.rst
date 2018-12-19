..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================
Reworking Prometheus Collector
======================================

CloudKitty's Prometheus collector needs some rework. This spec aims at defining
what is to be done and why. Everything in this document is open to discussion,
equally on the associated storyboard and gerrit.

Problem Description
===================

In the present state, Prometheus collector is not consistent with the other
collectors present in CloudKitty in terms of configuration and operations.

* The end user fully defines, on its own, the PromQL query to be executed
  in the ``metrics.yml`` file in a ``query`` option
  under ``extra_args`` subkey.
  Permitting the end user such complexity for building its request
  at the expense of genericity is not relevant regarding the use cases
  covered by CloudKitty.

* The collector does not use native PromQL ``groupby`` capabilities.
  The process is done post Prometheus service response causing
  an overhead operational cost.

* The collector ignores ``scope_key`` configuration option,
  disallowing scoping requests however it is the standard behaviour
  to be found in other collectors of the project.
  Besides, it is a major concern for orchestration.
  Let's say you deploy several CK processors. Without scoping,
  you will end up with data duplications likely to result in false reports.

The collector also needs the following new features:

* It needs authentication configuration options to dial into
  some secured Prometheus service.

* It needs a configuration option to take in account a custom certificate
  authority file in order to authorize HTTPS requests
  signed with custom certificate.

* It needs an insecure configuration option to explicitly trust HTTPS
  requests signed with an untrusted certificate.

These features will be useful to end users for configuring CloudKitty
to effectively connect to a Prometheus service.

Proposed Change
===============

In ``cloudkitty.conf``, under ``[collector_prometheus]`` section:

* Add a ``prometheus_user`` and a ``prometheus_password`` option
  to provide credentials for authentication (both defaults: none).

* Add a ``cafile`` option to allow custom certificate authority
  file (default: none).

* Add an ``insecure`` option to explicitly allow insecure HTTPS requests
  (default: false).


Possible deprecation of ``extra_args/query`` in ``metrics.yml``
for a defined metric (see Alternatives_ section).


In ``metrics.yml``, for a defined metric, add an ``aggregation_method`` option
under ``extra_args`` subkey to define which aggregation method to use collecting
metric data.

For instance:

.. code-block:: YAML

  # metrics.yml

  metrics:
    # [...]

    volume_size:
      unit: GiB
      groupby:
        - id
        - project_id
      metadata:
        - volume_type
      extra_args:
        aggregation_method: max

Valid aggregation methods will be the following :

* ``avg``: the average value of all points in the time window.
* ``min``: the minimum value of all points in the time window.
* ``max``: the maximum value of all points in the time window.
* ``sum``: the sum of all values in the time window.
* ``count``: the count of all values in the time window.
* ``stddev``: the population standard deviation of the values in the time window.
* ``stdvar``: the population standard variance of the values in the time window.

.. note::

   Time window is computed from the ``period`` configuration option under
   ``[collect]`` section in ``cloudkitty.conf``.

Other changes at this stage are mostly focused on the query building process
happening inside the Prometheus collector:

* ``[collect]/scope_key`` option value will be added to filter every query
  with the corresponding ``scope_id``.

* The collector will now use the ``groupby`` capabilities offered by PromQL
  using the syntax ``by (scope_key, groupby, metadata)`` instead of
  deferring it to the collector implementation.

Example
-------

For the following ``cloudkitty.conf`` :

.. code-block:: INI

   # Some options have been omitted for the sake of clarity

   [collect]
   collector = prometheus
   scope_key = namespace
   period = 3600

   [collector_prometheus]
   prometheus_url = http://prometheus-dn.tld/api/v1

And the following ``metrics.yml`` :

.. code-block:: YAML

  metrics:
    # [...]

    # Let's say we want to rate the following metric
    container_memory_usage_bytes:
      unit: GiB
      groupby:
        - container_id
      metadata:
        - volume_type
      extra_args:
        aggregation_method: max


The PromQL request will look the following :

``max(max_over_time(container_memory_usage_bytes{namespace="foobar"}[3600s])) by (namespace, container_id, volume_type)``

Start and end timestamps will be computed from the ``period`` option.
Let's say that we are starting from ``January 30th, 2019 at 1pm
(timestamp: 1548853200)``.
Then the end timestamp will be ``1548853200 + period (= 1548856800)``.

The HTTP URL with the url-encoded PromQL will then look the following :

``http://prometheus-dn.tld/api/v1/query?query=max%28max_over_time%28container_memory_usage_bytes%7Bnamespace%3D%22foobar%22%7D%5B3600s%5D%29%29%20by%20%28namespace%2C%20container_id%2C%20volume_type%29&time=158856800``


Alternatives
------------

Instead of deprecation for the ``extra_args/query`` option
defined for a metric in ``metrics.yml`` file, we could remove it.

Data model impact
-----------------

None

REST API impact
---------------

None

Security impact
---------------

None

Notifications Impact
--------------------

None

Other end user impact
---------------------

End users will probably have to reconfigure their metrics definition
if they are using the Prometheus collector in the present state.

However, after these changes are made,
they will have less overhead configuration to do in order to
switch from a collector to another, due to the consistency improvement
in the way metrics are defined in ``metrics.yml`` for Prometheus collector.

They will also benefit from the added configuration options
for the ``[collector_prometheus]`` section in ``cloudkitty.conf``.

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

Justin Ferrieu is assigned to work on the spec as well as to develop
the evoked points.

Primary assignee:
  jferrieu

Other contributors:
  None

Work Items
----------

* Change configuration schema, query building process and query response
  formatting process accordingly inside Prometheus collector.
* Add HTTPS and authentication support.

Dependencies
============

None

Testing
=======

The proposed changes will be tested with unit tests.

Documentation Impact
====================

We will add an entry for detailing the Prometheus collector configuration
in ``Administration Guide/Configuration Guide/Collector/Prometheus``.

References
==========

* Prometheus documentation: https://prometheus.io/docs/

