..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================================
 Adding some concurrency/parallelism to the processor
======================================================

The processor is currently single-threaded (except for AMQP listeners) and
running on a single process. This is a proposal to add some concurrency to
the processor, which will lead to a huge performance improvement.

https://storyboard.openstack.org/#!/story/2005423

Problem Description
===================

Given that the processor spends most of its time waiting once it has caught up
with the current timestamp, this is not a critical feature. However, this does
not imply much changes to the code, and would lead to a huge performance
improvement.

Having faster processors allows to catch up with the current timestamp
quicker, which is useful on new deployments, or when a long period needs to be
re-processed.

Proposed Change
===============

This change can be split into two parts. The first (and simplest) part would be
to retrieve all metrics for a given scope and period (pseudo-)simultaneously,
by making use of eventlet greenthreads. CloudKitty already depends on eventlet,
so this requires no new dependency. Once support for python2 will have been
dropped, this part can be updated in order to use the STL, and remove the
dependency on eventlet (which could be replaced by concurrent.futures or
asyncio).

The change is rather straightforward: rather than making consecutive calls
to ``_collect`` (one per metric type) in the workers, these should be made
simultaneously.

The second part would be to spawn several workers for each cloudkitty-processor
instance. For this, the proposal would be to use the **cotyledon** library,
which is already used by other OpenStack services, like ceilometer. Again, the
change is quite small: Instead of inheriting ``object``, the ``Orchestrator``
would inherit from ``cotyledon.Service``. A ``cotyledon.ServiceManager``
spawning several orchestrator services will also be added.

In order to limit the load on the network and memory, the number of workers
will be configurable.

Alternatives
------------

* Instead of using cotyledon, the STL's multiprocessing could be used. However,
  cotyledon has some useful features, like forwarding signals to the workers.
  Moreover, some new kind of services may be added in the future (rating
  agents etc...); cotyledon would allow to easily intergrate those without
  having to make much changes to the code.

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

None, apart from a notable performance improvement.

Performance Impact
------------------

A significant improvement to the processor's performance will be made.

Other deployer impact
---------------------

None

Developer impact
----------------

This adds a new dependency to cloudkitty: **cotyledon**. Introducing
concurrency and parallelism will encourage developpers to adopt a functional
coding style, in order to avoid race issues.

Implementation
==============

Assignee(s)
-----------

Gerrit topic: ``adding-concurrency``.

Primary assignee:
  peschk_l

Work Items
----------

* Retrieve metrics in eventlet greenthreads.

* Make cloudkitty-processor run several workers with cotyledon.

* Add an **orchestration** section to the documentation. It will contain
  details about how to configure the number of workers and how to configure
  the coordination URL.

Dependencies
============

* ``cotyledon``

Testing
=======

This will be tested by tempest scenarios, once these are implemented.

Documentation Impact
====================

An **orchestration** section will be added to the documentation. It will
contain details about how to configure the number of workers and how to
configure the coordination URL.

References
==========

* Cotyledon documentation: https://cotyledon.readthedocs.io/en/latest/index.html

* Eventlet documentation: https://eventlet.net/
