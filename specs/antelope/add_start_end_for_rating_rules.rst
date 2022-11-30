..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================================================
Set a time to live for rating rules on modules Hashmap and Pyscript.
====================================================================

This specification proposes an extension in the rating rules to add a life
time to the rules and add audit.


Problem Description
===================

In a dynamic cloud environment where computing resources are constantly
replaced to improve the cloud platform, we see the computing resources
prices being modified with the cloud environment upgrades; meaning, old
resources become cheaper, and users can choose the newest available
resources which usually are more expensive.

For that purpose, the price of computing resources is continually changing,
which means that they can change every year, or even more often according to
the cloud upgrades.

Nowadays in CloudKitty when we create rating rules (for both hash mapping and
Pyscripts), the rules are applied immediately after they were created, and
they keep working until they are deleted; the update of rating rules is also
possible, and we can do it whenever we want.

To keep the prices of the resources updated, operators need to change the cost
of each rating rule created, which causes some headaches when some are
missed/forgotten to be updated when they should. This generated the need to
execute data reprocessing. Besides that, as the rating rules updates take
effect right when they are updated, operators have to work at a specific
moment in time, which is not something very practical. For instance, if the
price changes at the very beginning of the year, which would force somebody
to update the rules at midnight of January of the new year, and so on.

Proposed Change
===============

To facilitate operators' work and make the Cloudkitty rating rules creation
process more dynamic and flexible, we propose to add a start and end date in
the hashmap and Pyscript rating rules modules.

Furthermore, as we are working with billing data, which will be used to charge
the cloud environment users, we also propose to create an auditing mechanism
for the rating rules; to achieve that, we will stop deleting the rating rules
from the database; instead of it, we will mark them as deleted and at the same
time, we will start denying the rule updating process in some specific
situations. The goal of these changes is to make the system more secure,
auditable, and predictable. Moreover, we will add three new columns, one to
store the user that created the rule, one with the user that
removed it, and the last one with the user that updated it.

The update is only going to be allowed if the rules have never been used
to rate a metric by CloudKitty. If it has already been used, we will not
allow updating it. The rationale is that we cannot change a rule while it
is being used, so we maintain it consistent. Operators will always be able
to delete a rating rule to be able to remove it from processing and
reprocessing. If rating rules have None as the end date, we then allow an
update (for rules that have not been used yet), which is the only update
that can be set for rating rules that are in use.

For rating rules that have not been used yet, we allow operators to update
anything they wish, including price, start, and end dates.

Another process that we will allow operators to do is to create rating rules
that start in a past time. As long as they send an extra parameter saying that
they understand that the data point that is in the past will not have this
rule to be applied if a reprocessing is not executed.

Database changes
================

Here are presented the proposed changes in Cloudkitty database schema.

Add new columns in the tables `hashmap_mappings` and `pyscripts_scripts`:

    - created_at: timestamp defining when the rule is created;

    - start: timestamp defining when the rule starts to be applied
      by CloudKitty;

    - end: timestamp defining when the rule ends/finishes/is not used
      anymore by CloudKitty. If NULL is used, it means endless;
      in other words, it is always applied;

    - name: the rating rule name which identifies the rule;

    - description: a rating rule description;

    - deleted: deletion timestamp;

    - created_by: keystone user's id who created the rule;

    - updated_by: last keystone user's id who changed the rule;

    - deleted_by: keystone user's id who deleted the rule.

.. code-block:: SQL

  +---------------------+--------------+------+---------------------+
  | Field               | Type         | Null | Default             |
  +---------------------+--------------+------+---------------------+
  | created_at          | timestamp    | NO   | CURRENT_TIMESTAMP   |
  | start               | timestamp    | NO   | CURRENT_TIMESTAMP   |
  | end                 | timestamp    | YES  |                     |
  | name                | varchar(32)  | NO   |                     |
  | description         | varchar(256) | YES  |                     |
  | deleted             | timestamp    | YES  |                     |
  | created_by          | varchar(32)  | NO   |                     |
  | updated_by          | varchar(32)  | YES  |                     |
  | deleted_by          | varchar(32)  | YES  |                     |
  +---------------------+--------------+------+---------------------+

REST API
========

Here are presented the proposed changes in Cloudkitty REST API.

  - path: /v1/rating/module_config/hashmap/mappings

    method: GET

    changes:

      - Filter by rating rule time to live window;
      - Filter by user who created/updated/deleted the rules;
      - Filter by description;
      - Show or not, via a flag, the deleted rating rules (marked
        as deleted with the deletion date);
      - Show or not, via a flag, the active rating rules (start and
        end dates period containing the current date);

  - path: /v1/rating/module_config/hashmap/mappings

    method: POST

    changes:

      - Four new values were added when creating rules:

          start: the timestamp the rule will take effect; if none is
          provided, the current timestamp will be used; if a date
          without time is provided, the midnight time (00:00:00)
          will be used; if the timezone is not provided, we use
          the timezone of the system. Its value must be < than
          the provided end date and cannot be in the past;

          end: the timestamp the rule will have no more effect; if none
          is provided, it will be none, meaning that it is an
          endless rule (rule without an end date); if a date without
          time is provided, the next midnight day minus 1 minute
          (23:59:00) will be used; if the timezone is not provided,
          we use the timezone of the system. Its value must be
          > than the provided start and cannot be in the past;

          name: the rule name, it is a required value, must be a string.
          The name must be unique for all not deleted rules (not
          marked as deleted);

          description: the rule description, is a optional parameter, and
          must be a string;

          force: optional value to allow users to create rules with a start
          and end in the past, for reprocessing purposes;

      - A new value will be processed in background when creating rules:

          created_by: We will set this value with the Keystone's authentication
          token used in the request, we will get the user id from the token and
          set it in this field;

  - path: /v1/rating/module_config/hashmap/mappings

    method: PUT

    changes:

      - We will allow to update only the rule's end date, nothing else,
        and only if the current end date is not defined (none). The new end
        value must be something in the future.
      - It will be also allowed to update rules where the start date is still
        in the future, for that case, the attributes allowed to be updated
        will be the:

          - start: if the provided one is in the future and < than end;
          - end: if the provided one is in the future and > than start;
          - cost;
          - description;

      - A new value will be processed in background when updating rules:

          updated_by: We will set this value with the Keystone's authentication
          token used in the request, we will get the user id from the token and
          set it in this field;

  - path: /v1/rating/module_config/hashmap/mappings

    method: DELETE

    changes:

      - We will not delete a rule anymore, instead, this endpoint will
        mark a rule as deleted, using the current date the request was called.

      - A new value will be processed in background when deleting rules:

          deleted_by: We will set this value with the Keystone's authentication
          token used in the request, we will get the user id from the token and
          set it in this field;

  - path: /v1/rating/module_config/pyscripts/scripts

    method: GET

    changes:

      - Filter by rating rule time to live window;
      - Filter by user who created/updated/deleted the rules;
      - Filter by description;
      - Show or not, via a flag, the deleted rating rules (marked
        as deleted with the deletion date);
      - Show or not, via a flag, the active rating rules (start and
        end dates period containing the current date);

  - path: /v1/rating/module_config/pyscripts/scripts

    method: POST

    changes:

      - Four new values were added to request when creating rules:

          start: the timestamp the rule will take effect; if none is
          provided, the current timestamp will be used; if a date
          without time is provided, the midnight time (00:00:00)
          will be used; if the timezone is not provided, we use
          the timezone of the system. Its value must be < than
          the provided end date and cannot be in the past;

          end: the timestamp the rule will have no more effect; if none
          is provided, it will be none, meaning that it is an
          endless rule (rule without an end date); if a date without
          time is provided, the next midnight day minus 1 minute
          (23:59:00) will be used; if the timezone is not provided,
          we use the timezone of the system. Its value must be
          > than the provided start and cannot be in the past;

          name: the rule name, it is a required value, must be a string.
          The name must be unique for all not deleted rules (not
          marked as deleted);

          description: the rule description, is a optional parameter, and
          must be a string;

          force: optional value to allow users to create rules with a start
          and end in the past, for reprocessing purposes;

      - A new value will be processed in background when creating rules:

          created_by: We will set this value with the Keystone's authentication
          token used in the request, we will get the user id from the token and
          set it in this field;

  - path: /v1/rating/module_config/pyscripts/scripts

    method: PUT

    changes:

      - We will allow to update only the rule's end date, nothing else,
        and only if the current end date is not defined (none). The new end
        value must be something in the future.
      - It will be also allowed to update rules where the start date is still
        in the future, for that case, the attributes allowed to be updated
        will be the:

          - start: if the provided one is in the future and < than end;
          - end: if the provided one is in the future and > than start;
          - cost;
          - description;

      - A new value will be processed in background when updating rules:

          updated_by: We will set this value with the Keystone's authentication
          token used in the request, we will get the user id from the token and
          set it in this field;

  - path: /v1/rating/module_config/pyscripts/scripts

    method: DELETE

    changes:

      - We will not delete a rule anymore, instead, this endpoint will
        mark a rule as deleted, using the current date the request was called.

      - A new value will be processed in background when deleting rules:

          deleted_by: We will set this value with the Keystone's authentication
          token used in the request, we will get the user id from the token and
          set it in this field;


Workflows
=========

Here are presented the proposed changes in the processing and reprocessing
workflows in Cloudkitty orchestrator.

Processing
==========

  - We will have three more filters (besides the tenant_id) when checking
    which rules must be used to process the data frame cost:

      - check if the rule's start timestamp is <= than the current timestamp;
        and
      - check if the rule's end timestamp is > than the current timestamp;
        and
      - check if there is no timestamp set in the deleted field of the rule;

Besides the changes introduced in this proposal, which we will filter the rules
processed, the rest of the processing workflow will stay the same.

Reprocessing
============

  - We will have three more filters (besides the tenant_id) when checking
    which rules must be used to process the data frames cost:

      - check if the rule's start timestamp is <= than the processed data frame timestamp;
        and
      - check if the rule's end timestamp is > than the processed data frame timestamp;
        and
      - check if there is no timestamp set in the deleted field of the rule;

Besides the changes introduced in this proposal, which we will filter the rules
processed, the rest of the reprocessing workflow will stay the same.

Final thoughts
==============

With this extension, if one needs to update any rule that has already been in use
(the start value is in the past), we will need to delete it and create a new one
with the updated values; alternatively, one can set the end date, and then create
a new one. By using this approach we can track all changes a rule has, which
facilitated auditing the system.

If all rules get expired, then we will start rating all the resources
with 0 (zero) value. However, that would be expected, as there is no valid rating
rules anymore. To avoid that, one can always use rules without and end date.

Alternatives
------------

Create some cron triggers to update the rating rules cost in the server;


Data model impact
-----------------

New columns will be added in the tables `hashmap_mappings` and
`pyscripts_scripts`;


REST API impact
---------------

New fields will be added in the
`/v1/rating/module_config/hashmap/mappings` and
`/v1/rating/module_config/pyscripts/scripts` endpoints

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
 - Pedro Henrique Pereira Martins <phpm13@gmail.com>


Work Items
----------

1) Extend the database model;
2) Create the data migration process to fill the initial start dates;
3) Adapt the REST APIs to accept the new fields;
    3.1) Adapt Hashmap mappings REST API;
    3.2) Adapt Pyscript REST API;
4) Create the validation mechanism to the new fields;
5) Change the deletion REST API to not delete the rules anymore;
6) Adapt the processing workflow to the new rules/validations;
7) Adapt the reprocessing workflow to the new rules/validations;
8) Change Cloudkitty CLI to accept new fields.

Dependencies
============
None

Testing
=======
Unit tested

Documentation Impact
====================
None

References
==========
None
