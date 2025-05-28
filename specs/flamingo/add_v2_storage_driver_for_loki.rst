..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================================================
Add v2 Storage Driver for Loki
====================================================================

This specification proposes Loki as a new storage backend for dataframes.

Problem Description
===================

CloudKitty currently offers robust integration with several storage backends,
namely Elasticsearch, OpenSearch, and InfluxDB. While these systems are
powerful and suitable for various data storage and querying needs, their
operational characteristics and resource footprints present challenges for
certain deployment scenarios and user ecosystems.

Loki has been positioning itself as a strong contender on the log storage and
search in container-based ecosystems. Its relatively easier operational
procedures, native container-orchestration support and similarity with
Prometheus, it seems to start becoming the preferred choice in many new
Kubernetes/OpenShift deployments. OpenShift even made it part of their default
Log Storage offering.

From a technical point of view, it is a very good system for storing
dataframe-like data: short pieces of json with dynamic fields and easily and
quickly searchable. Loki is a great system for ingest and search with very good
performance this kind of data: it is the exact problem it has been designed to
solve.


Proposed Change
===============

The proposed change will add a new v2 storage backend option for dataframes,
that will achieve feature parity with OpenSearch, but based on Loki. This means
that it will support the complete set of v1 and v2 APIs but will not have
support for custom_fields in the summary API.

The change will include unit tests, a new Zuul job with tempest tests and a
Loki devstack plugin for CI and development needs.

Final thoughts
==============

By supporting Loki as a storage we will be keeping CloudKitty updated and on
par with the more recent tendencies in the observability ecosystem, giving the
users a new choice, making CloudKitty easier to adopt if they are already
operating Loki for their log storage.

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
None

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
 - Juan Larriba <jlarriba@redhat.com>

Work Items
----------

1) Create a new Loki folder to contain the files for the new driver
2) Write a LokiClient class that abstracts away the Loki API interactions
3) Define the conf parameters that the new storage system is accepting
4) Write a LokiStorage class that extends BaseStorage and uses LokiClient to
   extract data from Loki and formats it in DataFrames
5) Write unit tests for LokiClient
6) Extend the TestStorageUnit with a new FakeLokiClient mock class
7) Change setup.cfg to include loki as a new storage option in
   cloudkitty.storage.v2.backends
8) Modify current documentation to add the Loki option as a new storage
   backend, highlighting the new parameters
9) Develop a new Loki devstack plugin so we can deploy Loki in the CI
10) Write a new CI cloudkitty-tempest-full-v2-storage-loki Zuul job to
    automatically test Loki integration

Dependencies
============

None

Testing
=======

* Unit tested
* Functionally tested using a Loki devstack plugin and tempest

Documentation Impact
====================

Documentation needs to be modified to state that there is a new storage backend
option available and describe the new parameters that are specific to the new
option.

References
==========

None
