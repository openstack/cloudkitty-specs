..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================================================
Adding some Prometheus query functions to the Prometheus collector
==================================================================

https://storyboard.openstack.org/#!/story/2006427

Problem Description
===================

The Prometheus collector cannot handle ``Counter`` and ``Gauge`` data types.

A ``Counter`` is a cumulative metric that represents a single monotonically
increasing counter whose value can only increase or be reset to zero on
restart. For example, you can use a counter to represent the number of
requests served, tasks completed, or errors.

A ``Gauge`` is a metric that represents a single numerical value that can
arbitrarily go up and down.

Proposed Change
===============

If the delta function was used, it would be possible to get the difference
between two collect cycles, and therefore to charge metrics of the
``Counter`` and ``Gauge`` type.

Adding two fields (``range_function`` and ``query_function``) to the
``extra_args`` section of the ``metrics.yml`` file would solve this problem.

Prometheus functions can be grouped in two categories, each matching one of
the new fields:

Some functions take a ``Range Vector`` as argument and some take
an ``Instant vector``. Both categories return an ``Instant vector``.

The new fields would only have a limited set of allowed functions
because of the expected result format.

The following Prometheus query functions will be allowed:

Functions taking a ``Range Vector``:

- ``changes()``
- ``delta()``
- ``deriv()``
- ``idelta()``
- ``irange()``
- ``irate()``
- ``rate()``

Functions taking an ``Instant Vector``:

- ``abs()``
- ``ceil()``
- ``exp()``
- ``floor()``
- ``ln()``
- ``log2()``
- ``log10()``
- ``round()``
- ``sqrt()``


Functions accepting a ``Range vector`` will be will be set through the new
``range_function`` field. They will replace the current implicit
``{aggregation_method}_over_time`` function that is applied by the collector
to obtain an ``Instant vector``. The field will be optional and will default to
``{aggregation_method}_over_time``.

Functions accepting an ``Instant vector`` will be set through the new
``query_function`` field. This will allow to apply an extra transformation to
data before it is rated.

Both fields can be combined.

Example
-------

The following config

.. code-block:: yaml

   metrics:
     gateway_function_invocation_total:
       unit: total
       groupby:
         - function_name
         - code
       extra_args:
         aggregation_method: max
         range_function: delta

would result in this query:
``max(delta(gateway_function_invocation_total{}[3600s])) by (function_name,code)``


Another example:

.. code-block:: yaml

   metrics:
     gateway_function_invocation_total:
       unit: total
       groupby:
         - function_name
         - code
       extra_args:
         aggregation_method: max
         range_function: delta
         query_function: abs

Would result in:
``max(abs(delta(gateway_function_invocation_total{}[3600s]))) by (function_name,code)``

And this:

.. code-block:: yaml

   metrics:
     gateway_function_invocation_total:
       unit: total
       groupby:
         - function_name
         - code
       extra_args:
         aggregation_method: max
         query_function: abs

Would result in:
``max(abs(max_over_time(gateway_function_invocation_total{}[3600s]))) by (function_name,code)``

Alternatives
------------

The PyScript module could be used in some cases but it's a bit complex just for
some simple operations. It would also require to save the latest
state of the metric (in case of the delta function).

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

End users will be able to perform more operations on metrics retrieved by
the Prometheus collector especially on ``Gauge`` and ``Counter`` metrics.

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
  <aimbot31>

Work Items
----------

* Add support for the ``query_function`` field to the Prometheus collector

* Add support for the ``range_function`` field to the Prometheus collector

Dependencies
============

None

Testing
=======

The proposed changes will be tested with Unit Tests.

Documentation Impact
====================

An entry detailing the configuration of the new field will be added to :
``Admin/Configuration/Collector``.

References
==========

* Prometheus functions documentation: https://prometheus.io/docs/prometheus/latest/querying/functions/
* Prometheus type documentation: https://prometheus.io/docs/prometheus/latest/querying/basics/
