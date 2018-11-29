OpenShift Build Service
=======================


OSBS is a collection of tools, workflows and integration points that build and release layered container images.

OSBS hooks into Koji_ with the help of the `koji-containerbuild plugin
<https://github.com/release-engineering/koji-containerbuild>`_, and
uses `OpenShift builds <https://docs.openshift.org/latest/dev_guide/builds.html>`_ as Content Generators to produce layered images.

.. _Koji: https://pagure.io/koji

One can start an image build with ``fedpkg``, using the ``container-build``
subcommand:

.. code::

    $ fedpkg container-build


Table of Contents
-----------------

.. toctree::
    :maxdepth: 3

    admins
    users
    build_process
    build_parameters
    contributors
    reference_docs


Indices and tables
==================

* :ref:`genindex`
* :ref:`search`

