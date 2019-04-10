..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================
Adding Prometheus fetcher
======================================

https://storyboard.openstack.org/#!/story/2005427

CloudKitty needs a fetcher describing how to get scopes (formerly tenants) from
a Prometheus service.
This spec aims at defining what is to be done and why.
Everything in this document is open to discussion, equally on the associated
storyboard and gerrit.

Problem Description
===================

In the present state, CloudKitty does not implement any method for discovering
and fetching scope IDs from a Prometheus service and it needs one.

Proposed Change
===============

The fetcher will be configurable under a ``[fetcher_prometheus]`` section
in ``cloudkitty.conf``.

The configuration options will be the folllowing :

* A ``metric`` option to provide a metric name from which we can infer
  the scope ID (default: none).

* A ``scope_attribute`` option to provide the key referring the scope IDs
  (default: ``project_id``).

* A ``filters`` option as a key-value dictionary to filter out the discovery
  query response with some metadata (optional).

* A ``prometheus_url`` option to define the HTTP API endpoint URL of a
  Prometheus service.

* A ``prometheus_user`` and a ``prometheus_password`` option
  to provide credentials for authentication (both defaults: none).

* A ``cafile`` option to allow custom certificate authority
  file (default: none).

* An ``insecure`` option to explicitly allow insecure HTTPS requests
  (default: false).

The fetcher will request an aggregation on the specified ``metric``
using PromQL query and send it over HTTP to the specified
Prometheus service API instant query endpoint.
It will also group the result by the specified ``scope_attribute`` and
filter out the response if some metadata ``filters`` have been specified
in order to gather the scope_ids.


Example
-------

.. code-block:: ini

  # cloudkitty.conf

  # [...]

  [fetcher_prometheus]
  metric=container_memory_usage_bytes
  scope_attribute=namespace
  prometheus_url=https://prometheus-dn.tld/api/v1
  prometheus_user=foobar
  prometheus_password=foobar
  insecure=true
  filters=label1:foo,label2:bar


The PromQL request will look the following:

``max(container_memory_usage_bytes{label1="foo", label2="bar"}) by (namespace)``


The HTTP URL with the url-encoded PromQL will then look the following :

``http://prometheus-dn.tld/api/v1/query?query=max%28container_memory_usage_bytes%7Blabel1%3D%22foo%22%2C%20label2%3D%22bar%22%7D%29%20by%20%28namespace%29``


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

End users will benefit from new Prometheus fetcher to dynamically discover
scope IDs.

It will be particularly handy in conjunction with the Prometheus collector.

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

* Implement Prometheus fetcher (with unit tests)
* Add fetcher documentation

Dependencies
============

None

Testing
=======

The proposed changes will be tested with unit tests.

Documentation Impact
====================

We will add an entry detailing the Prometheus fetcher configuration
in ``Administration Guide/Configuration Guide/Fetcher/Prometheus``.

References
==========

* Prometheus documentation: https://prometheus.io/docs/
