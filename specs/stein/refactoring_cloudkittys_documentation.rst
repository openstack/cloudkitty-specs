..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================
Refactoring CloudKitty's documentation
======================================

https://storyboard.openstack.org/#!/story/2004179

CloudKitty's documentation needs to be refactored. This spec aims at proposing
a new content layout. It is of course subject to discussion, as well
on storyboard as on gerrit.

Problem Description
===================

Since Cloudkitty has been through major changes in the last releases, the
current documentation layout and content causes several problems:

* The layout is not intuitive: many people on IRC ask for information that is
  available in the documentation, but very hard to find.

* Cloudkitty's architecture as well as the role of each component (fetcher,
  collector, rating module, storage backend...) is not explained clearly enough.
  This is a problem for operators, as they don't know how CloudKitty works
  internally, which is an issue when troubleshooting.

* The configuration of metric collection is very poorly documented, leading most
  operators to stick to the default ``metrics.yml`` file, without knowing
  exactly what each entry means.

* The developer documentation is almost non-existent: It only contains
  information about the v2 storage interface. It should be extended to help
  contributors.

* The documentation also lacks an FAQ/troubleshooting part: Typical question
  is "I deployed Cloudkitty with Devstack, why is summary 0 ?".

Proposed Change
===============

The first thing in the documentation should be a very brief introduction about
what CloudKitty is and does. Concerning people reading CloudKitty's
documentation: they can be split into three groups; users, operators/admins,
and developers. In consequence, proposed top-level sections after the short
introduction would be the following:

* Common introduction: What's the matter/need Cloudkitty aims to solve ?
  What's possible ? What's not (yet) ? What will never be supported ?

  In addition to these questions, there should be a general introduction on the
  different components, their roles and how they interact.

* User documentation: How do I retrieve information through cloudkitty's API ?
  What kind of information can I get ?

* HTTP API reference. However, reference of the v2 API will be published to
  ``developer.openstack.org/api-ref`` once it is considered stable.

* Operator documentation: This section should detail the role of each component,
  and provide information about how to configure it. This is also where
  information on rating rule creation should live. Operators are also the target
  audience for the troubleshooting section.

  In opposition to the current approach (with keystone/openstack vs.
  without keystone/openstack), I believe that this part of the documentation
  should explain each component one by one: its purpose, how it can be used,
  existing implementations (list different collectors/fetchers/rating modules,
  and ideally provide a support matrix), and how it can be configured.

  Example of a usecase where configuration is hard to deduce from the
  documentation: CloudKitty is used without keystone authentication and collects
  prometheus metrics. However, in order to collect metrics from gnocchi, the
  collector needs to authenticate to keystone. How to configure this ?

* Developer documentation: This section should provide implementation details
  about the different components cloudkitty is made of. It should explain
  contributors what they need to implement, which interfaces they should use,
  and how their contribution should be tested. Ideally, this should complete
  the docstrings of an interface. A (small and probably not complete enough)
  example of such a documentation can be found here:
  https://docs.openstack.org/cloudkitty/latest/developer/storage.html

  Here again, the documentation should be split by component.

Alternatives
------------

I don't have any alternative layouts or contents in mind, but proposals are
of course welcome!

Data model impact
-----------------
None

REST API impact
---------------

None, except maybe a better documentation of the API.

Security impact
---------------

None

Notifications Impact
--------------------

None

Other end user impact
---------------------

End users and contributors will have a better understanding of the project.

Performance Impact
------------------

None

Other deployer impact
---------------------

Better understanding of the projet, troubleshooting made easier.

Developer impact
----------------

Better understanding of the projet, contributing made easier.

Implementation
==============

Assignee(s)
-----------

Luka Peschke is assigned for the work on the spec, and the integration of the
new layout to the doc. Work on the content of the documentation will be
dispatched once the spec has been reviewed, corrected and merged.

Primary assignee:
  peschk_l

Other contributors:
  None

Work Items
----------

* Apply the new layout to cloudkitty's documentation

* Write an introduction with a few schemas

* Improve the documentation (sub-)section by (sub-)section, with one commit/task
  couple per subject.

Dependencies
============

None

Testing
=======

None

Documentation Impact
====================

The documentation will be completely modified, and a lot of content will be
added.

References
==========

* Glance documentation, which is a good example of a documentation organised per
  user profile: https://docs.openstack.org/glance/latest/

* Some support matrixes:

  - Elastic product / OS: https://www.elastic.co/support/matrix

  - Kubernetes persistent volumes access modes:
    https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes
