Documentation of the OSBS tools
===============================

Koji Containerbuild
-------------------

Extends Koji buildsystem with plugins to allow building containers
via OpenShift.

Provides plugin to submit builds with koji CLI.

* `Installation and usage instructions <https://github.com/release-engineering/koji-containerbuild/blob/master/README.rst>`__

* `General concepts behind container build architecture <https://github.com/release-engineering/koji-containerbuild/blob/master/docs/build_architecture.txt>`_


Atomic Reactor
--------------

Python library with command line interface for building docker images.

* `Installation and Usage Instructions <https://github.com/containerbuildsystem/atomic-reactor/blob/master/README.md>`_
* `Building Base Images <https://github.com/containerbuildsystem/atomic-reactor/blob/master/docs/base_images.md>`_
* `Plugins <https://github.com/containerbuildsystem/atomic-reactor/blob/master/docs/plugins.md>`_
* `build.json <https://github.com/containerbuildsystem/atomic-reactor/blob/master/docs/build_json.md>`_


OSBS Client
-----------

Python module and command line client for OpenShift Build Service.

* `Basic configuration and usage <https://github.com/containerbuildsystem/osbs-client/blob/master/README.md>`_
* `Description of the Build Process <https://github.com/containerbuildsystem/osbs-client/blob/master/docs/build_process.md>`_
* `Configuration file <https://github.com/containerbuildsystem/osbs-client/blob/master/docs/configuration_file.md>`_


dockpulp
--------

ReST API Client to Pulp for manipulating docker images.

* `Installation and Usage <https://github.com/release-engineering/dockpulp/blob/master/README.md>`_


Ansible Playbook
----------------

* `Ansible playbook for deploying OpenShift Build Service <https://github.com/projectatomic/ansible-osbs/blob/master/README.md>`_


Deploying and Running OSBS
--------------------------

* `Deploying OpenShift Build System <https://github.com/containerbuildsystem/osbs-client/blob/master/docs/osbs_instance_setup.md>`_
* `OSBS Backups <https://github.com/containerbuildsystem/osbs-client/blob/master/docs/backups.md>`_
* `Local Development <https://github.com/containerbuildsystem/osbs-client/blob/master/docs/development-setup.md>`_
* `Resource limiting <https://github.com/containerbuildsystem/osbs-client/blob/master/docs/resource.md>`_
