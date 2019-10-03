..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================
Optimize driver loading in v2 API
=================================

https://storyboard.openstack.org/#!/story/2006654

Problem Description
===================

Currently, required drivers are loaded in the ``__init__`` method of v2 API
resources. However, ``flask_restful`` instanciates resources on
`each request`_, meaning that drivers are also loaded on each request. This
leads to sub-optimal performance.

.. _`each request`: https://github.com/flask-restful/flask-restful/blob/20b220e9c4690a7a25474b6521ead342aaed0235/flask_restful/__init__.py#L465

Proposed Change
===============

Implement two new classmethods in ``cloudkitty.api.v2.base.BaseResource``:

* ``reload``: Reloads all required drivers.

* ``load``: Loads all required drivers by calling ``reload``. In case the
  drivers are already loaded, does nothing

In order to be able to persist drivers between class instantiations, they will
be set as class attributes.

Alternatives
------------

None.

Data model impact
-----------------

None.

REST API impact
---------------

None.

Security impact
---------------

None.

Notifications Impact
--------------------

None.

Other end user impact
---------------------

The average response time for v2 API endpoints will be reduced.

Performance Impact
------------------

Loading the drivers only once will result in a performance gain.

Other deployer impact
---------------------

Log messages that appear on driver instantiation will appear only once rather
than on each request.

Developer impact
----------------

Driver loading will have to be done in the ``reload`` method rather than in
``__init__``.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  <lukapeschke/peschk_l>

Work Items
----------

* Implement the ``reload`` and ``load`` methods.

Dependencies
============

None.

Testing
=======

Existing unit tests for the v2 API along with planned tempest tests will test
this.

Documentation Impact
====================

The developer documentation for the v2 API will be updated to include
information about the ``load`` and ``reload`` methods

References
==========

None.
