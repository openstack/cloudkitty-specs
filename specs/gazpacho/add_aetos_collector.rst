..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================
Add Aetos collector
======================================

Create a new Aetos collector to be able to access metrics exposed through the
Aetos reverse-proxy

Problem Description
===================

Aetos allows deployers to put Keystone authentication in front of Prometheus.
It's possible for deployers to want to have their Prometheus metrics fully
secure by not allowing direct unauthenticated access to Prometheus. This can
be done by allowing access to metrics only through Aetos, which requires
authentication through Keystone. In order for the deployer to be able
to do that and to use CloudKitty at the same time, CloudKitty needs to be able
to communicate with Aetos. That means, CloudKitty needs to be able to
authenticate itself when communicating with Aetos.

Proposed Change
===============

I propose to add a new Aetos client and collector. Because Aetos is just
adding an authentication layer to Prometheus, we can reuse most of the code we
currently have for Prometheus.

AetosClient implementation
--------------------------

First, a new cloudkitty/common/prometheus_client_base.py file will be created.
This file will contain a PrometheusClientBase class with _get and
get_instant abstract methods. The PrometheusResponseError exception
class will be moved into this file as well.

The PrometheusClient class in cloudkitty/common/prometheus_client.py
will now inherit from PrometheusClientBase. Otherwise, implementation
of this client will stay unchanged.

A new AetosClient class in cloudkitty/common/aetos_client.py will
be created, which will be a new concrete class inheriting from the
PrometheusClientBase. It'll use the python-observabilityclient to
create a client for Aetos and to implement the _get and get_instant methods.
A similar client setup already exists in Watcher
https://github.com/openstack/watcher/blob/master/watcher/decision_engine/datasources/aetos.py#L51

AetosCollector implementation
-----------------------------

Similarly to the client implementation, a new PrometheusCollectorBase class
will be created in cloudkitty/collector/prometheus_base.py. This class will
include the current PrometheusCollector code except for the __init__ method,
which will be implemented differently for each concrete class.
The PROMETHEUS_EXTRA_SCHEMA will move into prometheus_base.py as well.

The PrometheusCollector class from cloudkitty/collector/prometheus.py
will now inherit from PrometheusCollectorBase. Since most of the
current code will move into PrometheusCollectorBase class, the
PrometheusCollector class will implement only the __init__ method
and it'll set the collector_name. Both, the collector_name and the __init__
method, will stay unchanged.

A new cloudkitty/collector/aetos.py file will be created. In this
file, new Aetos related configuration options will be defined
(see below for their further description). A new AetosCollector
class will be created in this file as well, this class will inherit from
PrometheusCollectorBase, it'll set collector_name to 'aetos' and
it'll implement its own __init__ method, which will create an
AetosClient object based on configuration.

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

Access to metrics will be more secure with the Aetos collector.

Notifications Impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

The metric collection when using the Aetos collector will probably be a little
slower than with the current Prometheus collector. This is caused by having
an additional component (Aetos) in the path of accessing metrics.

Other deployer impact
---------------------

The Aetos collector will need to authenticate very similarly to how the
Gnocchi collector is doing it currently with keystone authentication,
which means it'll need similar config options.

The new [collector_aetos] section will have the following config
options:

interface
~~~~~~~~~

    Type:
        string
    Default:
        public
    Valid Values:
        internal, public, admin

    Type of endpoint to use in keystoneclient.

region_name
~~~~~~~~~~~

    Type:
        string
    Default:
        <None>

    Region in Identity service catalog to use for communication with the
    OpenStack service.

The [collect].collector option will have a possible new 'aetos' value.


Developer impact
----------------

This will add a new dependency 'python-observabilityclient'

Implementation
==============

Assignee(s)
-----------

- Jaromir Wysoglad
  IRC: jwysogla
  email: jwysogla@redhat.com

Work Items
----------

* Add implementation for the new Aetos client

* Add implementation for the new Aetos collector

* Add documentation for the new collector

* Add a zuul job for running CloudKitty with Aetos collector

Dependencies
============

New dependencies
----------------

python-observabilityclient - this is the client library used to access
metrics through Aetos.

Dependent work in other projects
--------------------------------

Aetos will need a slight expansion to its query/ endpoint first to allow
CloudKitty to use the 'time' and 'timeout' GET parameters like it's currently
doing when querying Prometheus. This doesn't have a spec and it's not being
tracked anywhere. A change for this will be worked on and submitted in
parallel with the Aetos collector implementation in CloudKitty.

Testing
=======

* Unit tests

* The cloudkitty-tempest-full-v2-fetcher-collector-prometheus job will be
  duplicated into cloudkitty-tempest-full-v2-fetcher-keystone-collector-aetos.
  This job will be configured to use keystone as the fetcher
  and aetos as the collector. The same set of tempest tests that are currently
  being used for Prometheus should be reused for Aetos, since Aetos
  is compatible with Prometheus except for the added authentication.

* The cloudkitty-tempest-full-v2-fetcher-collector-prometheus job will be
  added to Aetos's check and gate pipelines as well to ensure future changes to
  Aetos don't cause issues in CloudKitty.

Documentation Impact
====================

* A new "Aetos" section will be added to the "Collector configuration"
  documentation with description of the new config options in cloudkitty.conf
  https://docs.openstack.org/cloudkitty/latest/admin/configuration/collector.html#collector-specific-options

* Documentation of Prometheus specific options in metric.yml will be renamed
  to "Prometheus and Aetos", because metric.yml will be 100% compatible
  between Prometheus and Aetos collectors.
  https://docs.openstack.org/cloudkitty/latest/admin/configuration/collector.html#id2

* Aetos collector will be added to the list of collectors in Architecture
  documentation
  https://docs.openstack.org/cloudkitty/latest/admin/architecture.html#collector

References
==========

* Aetos: https://opendev.org/openstack/aetos

* python-observabilityclient: https://opendev.org/openstack/python-observabilityclient
