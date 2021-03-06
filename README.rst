============
python-aptly
============

Aptly REST API client and useful tooling

Publisher
=========

Publisher is tooling for easier maintenance of complex repository management
workflows.

This is how workflow can look like and what publisher can do for you:

.. image:: ./doc/aptly-publisher.png
    :align: center

Features
--------

- create or update publish from latest snapshots

  - it takes configuration in yaml format which defines what to publish and
    how
  - expected snapshot format is ``<name>-<timestamp>``

- promote publish

  - use source publish snapshots to create or update another publish (eg.
    testing -> stable)

- cleanup unused snapshots

Create or update publish
~~~~~~~~~~~~~~~~~~~~~~~~

First create configuration file where you define Aptly repositories, mirrors
and target distributions for publishing.

.. code-block:: yaml

    mirror:
      # Ubuntu upstream repository
      trusty-main:
        # Base for our main component
        component: main
        distributions:
          - nightly/trusty
      # Mirrored 3rd party repository
      aptly:
        # Merge into main component
        component: main
        distributions:
          - nightly/trusty

    repo:
      # Some repository with custom software
      cloudlab:
        # Publish as component cloudlab
        component: cloudlab
        distributions:
          # We want to publish our packages (that can't break anything for
          # sure) immediately to both nightly and testing repositories
          - nightly/trusty
          - testing/trusty

Configuration above will create two publishes from latest snapshots of
defined repositories and mirrors:

- ``nightly/trusty`` with component cloudlab and main

  - creates snapshot ``_main-<timestamp>`` by merging snapshots
    ``aptly-<timestamp>`` and ``trusty-main-<timestamp>``)

- ``testing/trusty`` with component cloudlab, made of repository cloudlab

It expects that snapshots are already created (by mirror syncing script or by
CI when new package is built) so it does following:

- find latest snapshot (by creation date) for each defined mirror and
  repository

  - snapshots are recognized by name (eg. ``cloudlab-<timestamp>``,
    ``trusty-main-<timestamp>``)

- create new snapshot by merging snapshots with same publish component

  - eg. create ``_main-<timestamp>`` from latest ``trusty-main-<timestamp>``
    and ``aptly-<timestamp>`` snapshots
  - merged snapshots are prefixed by ``_`` to avoid collisions with other
    snapshots
  - first it checks if merged snapshots already exists and if so, it will skip
    creation of duplicated snapshot. So it's tries to be fully idempotent.

- create or update publish or publishes as defined in configuration

It can be executed like this:

::

  aptly-publisher -c config.yaml -v --url http://localhost:8080 publish

Promote publish
~~~~~~~~~~~~~~~

Let's assume you have following prefixes and workflow:

- nightly

  - created by `publish` action when there's new snapshot or synced mirror
  - packages are always up to date

- testing

  - freezed repository for testing and stabilization

- stable

  - well tested package versions
  - well controlled update process

There can be more publishes under prefix, eg. ``nightly/trusty``,
``nightly/vivid``

Then you need to switch published snapshots from one publish to another one.

::

  aptly-publisher -v --url http://localhost:8080  \
  --source nightly/trusty --target testing/trusty \
  publish

You can also specify list of components. When you have separate components for
your packages (eg. cloudlab) and security (mirror of trusty security
repository), you may need to release them faster.

::

  aptly-publisher -v --url http://localhost:8080  \
  --source nightly/trusty --target testing/trusty \
  --components cloudlab security -- publish

Finally you are also able to promote selected packages, eg.

::

  aptly-publisher -v --url http://localhost:8080  \
  --source nightly/trusty --target testing/trusty \
  --packages python-aptly aptly -- publish

Show differencies between publishes
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

You can see differencies between publishes with following command:

::

  aptly-publisher -v --url http://localhost:8080  \
  --source nightly/trusty --target testing/trusty \
  publish --diff

Example output can look like this:

.. image:: ./doc/publisher_diff_example.png
    :align: center

Cleanup unused snapshots
~~~~~~~~~~~~~~~~~~~~~~~~

When you are creating snapshots regularly, you need to delete old ones that
are not used by any publish. It's wise to call such action every time when
publish is updated (eg. nightly).

::

  aptly-publisher -v --url http://localhost:8080 cleanup

Installation
============

You can install directly using from local checkout or from pip:

::

  python setup.py install
  pip install python-aptly


Or better build Debian package with eg.:

::

  dpkg-buildpackage -uc -us

Read more
=========

For usage informations, see ``aptly-publisher --help`` or man page.

::

  man man/aptly-publisher.1

Also see ``doc/examples`` directory.

For examples of jenkins jobs, have a look at `tcpcloud/jenkins-jobs <https://github.com/tcpcloud/jenkins-jobs>`_ repository.

Known issues
============

- determine source snapshots correctly
  (`#271 <https://github.com/smira/aptly/issues/271>`_)
- cleanup merged snapshots before cleaning up source ones

  - before that it's needed to run cleanup action multiple times to get all
    unused snapshots cleaned
