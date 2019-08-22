===================================
OpenStack Cloudkitty Specifications
===================================

This git repository is used to hold approved design specifications for additions
to the Cloudkitty project. Reviews of the specs are done in gerrit, using a
similar workflow to how we review and merge changes to the code itself.

The layout of this repository is::

  specs/<release>/

You can find an example spec in `specs/template.rst`.

Specifications are proposed for a given release by adding them to the
`specs/<release>` directory and posting it for review.  The implementation
status of a story for a given release can be found by looking at the
story in Storyboard. Not all stories will get fully implemented.

Specifications have to be re-proposed for every release. The review may be
quick, but even if something was previously approved, it should be re-reviewed
to make sure it still makes sense as written.

For more information about working with gerrit, see::

  https://docs.openstack.org/infra/manual/developers.html#development-workflow

To validate that the specification is syntactically correct (i.e. get more
confidence in the Jenkins result), please execute the following command::

  $ tox

After running ``tox``, the documentation will be available for viewing in HTML
format in the ``doc/build/`` directory.
