..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================
Add DataFrame/DataPoint objects
===============================

CloudKitty has an inner data format called "DataFrame". It is used almost
everywhere: The API returns dataframes, the storage driver expects to store
dataframes, collected data is retrieved as a dataframe... But dataframes are
always passed around as dicts, making their manipulation tedious. This is
a proposal to add a DataFrame and DataPoint class definition, which would
allow easier conversion/manipulation of dataframes.

https://storyboard.openstack.org/#!/story/2005890

Problem Description
===================

The "dataframe" format is specified in multiple places, but there is no true
implementation of it: dicts respecting the format specifications are passed
around instead. This can be error-prone: the integrity of these objects is not
guaranteed (a function might modify them, even without intending to), and some
specific details may vary from one part of the codebase to another (for
example a ``float`` may be used instead of a ``decimal.Decimal``).

Furthermore, the dataframe format is not exactly the same in the v1 and v2
storage interfaces. v1 has a single ``desc`` key containing every metadata
attribute of a data point, whereas v2 provides two keys, ``metadata`` and
``groupby``, depending on the type of the attribute. This leads to conversions
between v1 and v2 format in several places in the code. Example taken from the
``CloudKittyFormatTransformer``:

.. code-block:: python

   def format_item(self, groupby, metadata, unit, qty=1.0):
       data = {}
       data['groupby'] = groupby
       data['metadata'] = metadata
       # For backward compatibility.
       data['desc'] = data['groupby'].copy()
       data['desc'].update(data['metadata'])
       data['vol'] = {'unit': unit, 'qty': qty}

       return data

Proposed Change
===============

The proposed solution is to introduce two new classes: ``DataPoint`` and
``DataFrame``.

``DataPoint``
+++++++++++++

``DataPoint`` replaces a single data point represented by a dict with the
following format:

.. code-block:: python

   {
       "vol": {
           "unit": "GiB",
           "qty": 1.2,
       },
       "rating": {
           "price": 0.04,
       },
       "groupby": {
           "group_one": "one",
           "group_two": "two",
       },
       "metadata": {
           "attr_one": "one",
           "attr_two": "two",
       },
   }

The following attributes will be accessible in a ``DataPoint`` object:

* ``qty``: ``decimal.Decimal``
* ``price``: ``decimal.Decimal``
* ``groupby``: ``werkzeug.datastructures.ImmutableMultiDict``
* ``metadata``: ``werkzeug.datastructures.ImmutableMultiDict``
* ``desc``: ``werkzeug.datastructures.ImmutableMultiDict``

.. note:: ``desc`` will be a combination of ``metadata`` and ``groupby``

In order to ensure data consistency, the ``DataPoint`` object will inherit
``collections.namedtuple``. The ``groupby`` and ``metadata`` attributes will
be stored as ``werkzeug.datastructures.ImmutableDict``.

In addition to its base attributes, the ``DataPoint`` class will have a
``desc`` attribute (implemented as a property), which will return an
``ImmutableDict`` (a merge of ``metadata`` and ``groupby``).

``DataPoint`` instances will expose the following methods:

* ``set_price``: Set the price of the ``DataPoint``. Returns a new instance.

* ``as_dict``: Returns an (optionally mutable) dict representation of the
  object. For convenience with API backward compatibility, it will be possible
  to obtain the result in legacy format (``desc`` will replace ``metadata``
  and ``groupby``).

* ``json``: Returns a json representation of the object. For convenience with
  API backward compatibility, it will be possible to obtain the result in
  legacy format (``desc`` will replace ``metadata`` and ``groupby``).

* ``from_dict``: Creates a ``DataPoint`` from its dict representation.

``DataFrame``
+++++++++++++

``DataFrame`` replaces a dataframe represented by a dict with the following
format:

.. code-block:: python

   {
       "period": {
           "begin": datetime.datetime,
           "end": datetime.datetime,
       },
       "usage": {
           "metric_one": [], # list of datapoints
           [...]
       }
   }

A ``DataFrame`` is a wrapper around a collection of ``DataPoint`` objects.
``DataFrame`` instances will have two read-only attributes: ``start`` and
``end`` (stored as ``datetime.datetime`` objects).

``DataFrame`` instances will expose the following methods:

* ``as_dict``: Returns an (optionally mutable) dict representation of the
  object. For convenience with API backward compatibility, it will be possible
  to obtain the result in legacy format.

* ``json``: Returns a json representation of the object. For convenience with
  API backward compatibility, it will be possible to obtain the result in
  legacy format.

* ``from_dict``: Creates a ``DataFrame`` from its dict representation.

* ``add_points``: Adds a list of ``DataPoint`` objects to a dataframe for a
  given metric.

* ``iterpoints``: Generator function iterating over all points in the
  ``DataFrame``. Yields (metric_name, ``DataPoint``) tuples.

.. note:: Given that the ``from_dict`` method of both classes will mainly be
          used at the API level, voluptuous schemas matching the classes
          will be added and a schema validation will be executed on the
          argument ``from_dict`` is called with.


Alternatives
------------

The code-base could be left as is, letting developers deal with the tedious
dataframe manipulations.

Data model impact
-----------------

Data structures manipulated internally get hardened.

REST API impact
---------------

None. However, this would ease a future endpoint allowing to push dataframes
to cloudkitty.

Security impact
---------------

None.

Notifications Impact
--------------------

None.

Other end user impact
---------------------

None.

Performance Impact
------------------

Instantiating ``DataPoints`` might be slightly slower than instantiating dicts.
However, ``namedtuple`` is a high-performance container, and several
dict formatting steps that are currently executed will be skipped if we use
a ``namedtuple`` subclass, so there may be no overhead at all.

Other deployer impact
---------------------

None.

Developer impact
----------------

Manipulating objects with a clear and strict interface should make
developing with dataframes easier and way less error-prone.

No extra dependencies are required.

Implementation
==============

Assignee(s)
-----------

Primary assignee:

  peschk_l

Work Items
----------

* Create validation utils that will allow to check the datapoint/dataframe
  format.

* Submit the new classes along with tests.

Dependencies
============

None.

Testing
=======

This will be tested with unit tests. A 100% test coverage is expected.

Documentation Impact
====================

None.

References
==========

None.
