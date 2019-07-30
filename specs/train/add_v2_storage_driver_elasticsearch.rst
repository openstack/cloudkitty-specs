..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================
Add an Elasticsearch v2 storage driver
======================================

https://storyboard.openstack.org/#!/story/2006332

Problem Description
===================

For now, there is only one v2 storage driver: InfluxDB. However there should
always be several proposed choices for each of CloudKitty's modules.

The following strengths make Elasticsearch a great candidate:

* **It's a widespread solution.** On most deployments, it is likely that an
  Elasticsearch cluster is available (for example for log centralization).

* **HA and clustering**. Elasticsearch features HA and clustering by default,
  whereas it is only available in the paid version of InfluxDB.

* **It's performant.** And Elasticsearch allows some tuning by admins.

* **Data visualization** Data from the InfluxDB storage driver can be
  visualized with Grafana, the Elastic stack provides Kibana.

Proposed Change
===============

A v2 storage driver for Elasticsearch, available through the
``cloudkitty.storage.v2.backends.elasticsearch`` entrypoint.

Here's a summary of the routes and aggregation methods that will be used for
each of the v2 storage driver interface's methods:

* ``init``: ``PUT /<index>`` See "Data model impact" for mapping details.

* ``push``: ``POST /<index>/<mapping>/_bulk``

* ``retrieve``: ``GET /<index>/_search`` A standard search query with filters.

* ``total``: ``GET /<index>/_search`` The ``composite`` aggregation will be
  used: Several ``terms`` aggregations in the ``must`` clause of a ``bool``
  query will allow to group data on specific attributes.
  A ``sum`` aggregation will then be applied to the buckets to obtain the qty
  and price for each of them.

* ``delete``: ``POST /<index>/_delete_by_query`` Same principle as the
  ``retrieve`` method, but for deletion.


.. warning:: The "composite" query is stable since Elasticsearch version
             6.5. In order to be compatible with 6.x and 7.x, cloudkitty will
             use the ``include_type_name`` parameter for mapping creation. This
             parameter was added in Elasticsearch 6.8. This parameter will be
             removed in Elasticsearch 8. Thus, CloudKitty will require
             Elasticsearch >= 6.5 and < to 8.

.. note:: **About pagination:** Given that ``offset`` + ``size`` can't exceed
          15000 in the ``search`` API, the ``retrieve`` function will use
          scrolling. The ``search_after`` feature will not be used, as it
          is stateless, which means that consecutive requests may return
          unexpected results depending on the index updates happening at the
          same time. The duration for which scroll contexts should be kept
          open will be configurable through a config file option marked as
          **advanced**.

          The ``total`` function will use the ``after`` parameter of the
          ``composite`` aggregation

.. note:: The CloudKitty storage driver will only require the OSS version of
          Elasticsearch to work. However, some X-Pack features of the Basic
          version, like authentication, will be supported (but not mandatory)
          in the future.

Alternatives
------------

None.

Data model impact
-----------------

The data model used in Elasticsearch will be as follows:

Each DataPoint will be a single document. An existing empty index is required
(this will allow tuning from admins). In order to improve overall performance,
a mapping with the following attributes will be created.

- ``start``: (date) The start of the period the datapoint applies to.
- ``end``: (date) The end of the period the datapoint applies to.
- ``type``: (keyword) The type of the datapoint.
- ``unit``: (keyword) The unit of the datapoint.
- ``qty``: (double) The qty of the datapoint.
- ``price``: (double) The price of the datapoint.
- ``groupby``: (object) Dict of the datapoint's groupby attributes.
- ``metadata``: (object) Dict of the datapoint's metadata attributes.

.. note:: In order to allow flexible groupby/metadata, the associated objects
          will be flexible.

.. note:: Given that we will only do exact value searches, every ``string``
          attribute will be converted to a ``keyword``. This will be achieved
          using dynamic templates.

.. warning:: By default, the ``_source`` field will be enabled. An option to
             disable it in order to improve storage size may be added,
             but this should be done with care. See the link to the
             Elasticsearch documentation in the references for details.

In the end, the mapping will be defined as follows:

.. code-block:: javascript

   {
     "mappings": {
       "_doc": {
         // cast all strings to keywords
         "dynamic_templates": [
           {
             "strings_as_keywords": {
               "match_mapping_type": "string",
               "mapping": {
                 "type": "keyword"
               }
             }
           }
         ],
         // we won't add any attribute to the base object, so dynamic must be false
         "dynamic": false,
         "properties": {
           "start": {"type": "date"},
           "end": {"type": "date"},
           "type": {"type": "keyword"},
           "unit": {"type": "keyword"},
           "qty": {"type": "double"},
           "price": {"type": "double"},
           // groupby and metadata will accept new attributes
           "groupby": {"dynamic": true, "type": "object"},
           // even though metadata should not be indexed, disabling it can't be
           // undone, and disabled objects are only available through the "_source"
           // field, which may also be disabled
           "metadata": {"dynamic": true, "type": "object"}
         }
       }
     }
   }

.. note:: Given that a term to filter on may be part of ``groupby`` or
          ``metadata``, each filter will add two ``term`` queries to the
          ``should`` part of the ``bool`` query (one for the ``groupby``
          section and one for the ``metadata`` section). Thus, the
          ``minimum_should_match`` parameter of the ``bool`` query will be set
          to half of the number of terms in the ``should`` query.

REST API impact
---------------

None.

Security impact
---------------

In the first iteration, there will be no support for x-pack authentication.
It will be up to the admins to secure the connections between the Elasticsearch
cluster and CloudKitty. Authentication will be introduced in future releases.

Notifications Impact
--------------------

None.

Other end user impact
---------------------

None.

Performance Impact
------------------

On most benchmarks (and from what could be determined from POCs), data insertion
into Elasticsearch is slower than insertion into InfluxDB. However,
Elasticsearch is faster for aggreations. However, once CloudKitty has caught
up with the current timestamp, not many insertions are required. Moreover,
Elasticsearch's support for clustering and for tuning should allow for a
better overall perfomance in the end.

Other deployer impact
---------------------

The new backend will require more configuration from the admins:

* Index aliases and lifecycles

* Shards and replicas

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  peschk_l

Work Items
----------

* Implement an Elasticsearch storage driver
* Add support for the driver to the Devstack plugin.
* Add a Tempest job where the Elasticsearch storage driver is used.

Dependencies
============

Elasticsearch >= 6.5.

Testing
=======

In addition to unit tests, this will be tested with Tempest.

Documentation Impact
====================

The configuration options provided of this driver will be detailed in the
documentation. There will also be a section dedicated to the configuration of
the Elasticsearch index.

References
==========

* Dynamic templates: https://www.elastic.co/guide/en/elasticsearch/reference/current/dynamic-templates.html

* ``_source`` field: https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-source-field.html

* ``search_after`` parameter: https://www.elastic.co/guide/en/elasticsearch/reference/7.x/search-request-body.html#request-body-search-search-after

* Elasticsearch and InfluxDB benchmarks: https://jolicode.com/blog/influxdb-vs-elasticsearch-for-time-series-and-metrics-data
