Administration and Maintenance
==============================

Setting up koji for container image builds
------------------------------------------

The /etc/osbs.conf file used by the koji-containerbuild plugin must
name secrets configured on the OpenShift orchestration cluster:

reactor_config_secret
    This secret holds details about worker clusters; see
    :ref:`config.yaml`

client_config_secret
    This secret holds the osbs.conf which will be used by
    atomic-reactor to create worker builds; see
    :ref:`client_config_secret`

token_secrets
    This set of secrets may be referenced by the osbs.conf held in
    client_config_secret, for example for service account tokens on
    worker clusters; see :ref:`token_secrets`

Configuring osbs-client
-----------------------

Can Orchestrate
~~~~~~~~~~~~~~~

The parameter ``can_orchestrate`` defaults to false. The API method
``create_orchestrator_build`` will fail unless ``can_orchestrate`` is
true for the chosen instance section.

.. _reactor_config_secret:

Reactor config secret
~~~~~~~~~~~~~~~~~~~~~

When ``reactor_config_secret`` is specified this is the name of a
Kubernetes secret holding :ref:`config.yaml`. A pre-build plugin will
be configured with the location this secret is mounted.

.. _client_config_secret:

Client config secret
~~~~~~~~~~~~~~~~~~~~

When ``client_config_secret`` is specified this is the name of a
Kubernetes secret holding ``osbs.conf`` for use by atomic-reactor when it
creates worker builds. The `orchestrate_build` plugin is told the
path to this.

.. _token_secrets:

Token secrets
~~~~~~~~~~~~~

When ``token_secrets`` is specified the specified secrets (space
separated) will be mounted in the OpenShift build. When ":" is used,
the secret will be mounted at the specified path, i.e. the format is::

  token_secrets = secret:path secret:path ...

This allows an ``osbs.conf`` file (from ``client_config_secret``) to
be constructed with a known value to use for ``token_file``.

Deploy OSBS on OpenShift
------------------------

Authentication
~~~~~~~~~~~~~~

The orchestrator cluster will have a service account (with edit role)
created for use by Koji builders. Those Koji builders will use the
service account's persistent token to authenticate to the orchestrator
cluster and submit builds to it.

Since the orchestrator build initiates worker builds on the worker
cluster, it must have permission to do so. A service account should be
created on each worker cluster in order to generate a persistent
token. This service account should have edit role. On the orchestrator
cluster, a secret for each worker cluster should be created to store
the corresponding service account tokens. When osbs-client creates the
orchestrator build it must specify the names of the secret files to be
mounted in the BuildConfig. The orchestrator build will extract the
token from the mounted secret file.

.. _config.yaml:

Server-side Configuration for atomic-reactor
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This will list the maximum number of jobs that should be active at any
given time for each cluster. It will also list worker clusters in
order of preference.

The runtime configuration will take the form of a Kubernetes secret
with content as in the example below::

  clusters:
    x86_64:
    - name: prod-x86_64-osd
      max_concurrent_builds: 16
    - name: prod-x86_64
      max_concurrent_builds: 6
      enabled: true
    - name: prod-other
      max_concurrent_builds: 2
      enabled: false

    ppc64le:
    - name: prod-ppc64le
      max_concurrent_builds: 6

.. _config.yaml-clusters:

clusters
''''''''

This maps each platform to a list of clusters and their concurrent
build limits. For each platform to build for, a worker cluster is
chosen as follows:

- clusters with the enabled key set to false are discarded
  
- each remaining cluster in turn will be queried to discover all
  currently active worker builds (not failed, complete, in error, or
  cancelled)

- the cluster load is computed by dividing the number of active worker
  builds by the specified maximum number of concurrent builds allowed
  on the cluster

- the worker build is submitted to whichever cluster has the lowest
  load; in this way, an even load distribution across all clusters is
  enforced

There are several throttles preventing too many worker builds being
submitted. Each worker cluster can be configured to only schedule a
certain number of worker builds at a time by setting a default
resource request. The orchestrator cluster will similarly only run a
certain number of orchestrator builds at a time based on the resource
request in the orchestrator build JSON template. A Koji builder will
only run a certain number of containerbuild tasks based on its
configured capacity.

This mechanism can also be used to temporarily disable a worker
cluster by removing it from the list or adding ``enabled: false`` to
the cluster description for each platform.

Pushing built images to Pulp
----------------------------

