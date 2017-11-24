Administration and Maintenance
==============================

Configuring osbs-client
-----------------------

When submitting a new build request to OSBS as a user, this request is
for an orchestrator build. When the orchestrator build wants to create
worker builds it also does this through osbs-client.

As a result there are two osbs.conf files to consider:

- the one external to OSBS, for creating orchestrator builds, and
- the one internal to OSBS, stored in a Kubernetes secret (named by
  `client_config_secret`_) in the orchestrator cluster

These can have the same content. The important features are discussed
below.

can_orchestrate
~~~~~~~~~~~~~~~

The parameter ``can_orchestrate`` defaults to false. The API method
``create_orchestrator_build`` will fail unless ``can_orchestrate`` is
true for the chosen instance section.

reactor_config_secret
~~~~~~~~~~~~~~~~~~~~~

When ``reactor_config_secret`` is specified this is the name of a
Kubernetes secret holding :ref:`config.yaml`. A pre-build plugin will
be configured with the location this secret is mounted.

.. _client_config_secret:

client_config_secret
~~~~~~~~~~~~~~~~~~~~

When ``client_config_secret`` is specified this is the name of a
Kubernetes secret (holding a key ``osbs.conf``) for use by
atomic-reactor when it creates worker builds. The `orchestrate_build`
plugin is told the path to this.

token_secrets
~~~~~~~~~~~~~

When ``token_secrets`` is specified the specified secrets (space
separated) will be mounted in the OpenShift build. When ":" is used,
the secret will be mounted at the specified path, i.e. the format is::

  token_secrets = secret:path secret:path ...

This allows an ``osbs.conf`` file (from `client_config_secret`_) to
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

Setting up koji for container image builds
------------------------------------------

Example configuration file: Koji builder
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The configuration required for submitting an orchestrator build is
different than that required for the orchestrator build itself to
submit worker builds. The ``osbs.conf`` used by the Koji builder would
include::

  [general]
  build_json_dir = /usr/share/osbs/

  [platform:x86_64]
  architecture = amd64
  enable_v1 = true

  [default]
  openshift_url = https://orchestrator.example.com:8443/
  build_image = example.registry.com/buildroot:blue

  distribution_scope = public

  can_orchestrate = true  # allow orchestrator builds

  # This secret contains configuration relating to which worker
  # clusters to use and what their capacities are:
  reactor_config_secret = reactorconf

  # This secret contains the osbs.conf which atomic-reactor will use
  # when creating worker builds
  client_config_secret = osbsconf

  # These additional secrets are mounted inside the build container
  # and referenced by token_file in the build container's osbs.conf
  token_secrets =
    workertoken:/var/run/secrets/atomic-reactor/workertoken

  # and auth options, registries, secrets, etc

  [scratch]
  openshift_url = https://orchestrator.example.com:8443/
  build_image = example.registry.com/buildroot:blue

  reactor_config_secret = reactorconf
  client_config_secret = osbsconf
  token_secrets = workertoken:/var/run/secrets/atomic-reactor/workertoken

  # All scratch builds have distribution-scope=private
  distribution_scope = private

  # This causes koji output not to be configured, and for the low
  # priority node selector to be used.
  scratch = true

  # and auth options, registries, secrets, etc

This shows the configuration required to submit a build to the
orchestrator cluster using ``create_prod_build`` or
``create_orchestrator_build``.

Also shown is the configuration for scratch builds, which will be
identical to regular builds but with "private" distribution scope for
built images and with the scratch option enabled.

Example configuration file: inside builder image
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The ``osbs.conf`` used by the builder image for the orchestrator
cluster, and which is contained in the Kubernetes secret named by
``client_config_secret`` above, would include::

  [general]
  build_json_dir = /usr/share/osbs/

  [platform:x86_64]
  architecture = amd64
  enable_v1 = true

  [prod-mixed]
  registry_api_versions = v1,v2
  openshift_url = https://worker01.example.com:8443/
  node_selector.x86_64 = beta.kubernetes.io/arch=amd64
  node_selector.ppc64le = beta.kubernetes.io/arch=ppc64le
  use_auth = true

  # This is the path to the token specified in a token_secrets secret.
  token_file =
    /var/run/secrets/atomic-reactor/workertoken/worker01-serviceaccount-token

  # The same builder image is used for the orchestrator and worker
  # builds, but used with different configuration. It should not
  # be specified here.
  # build_image = registry.example.com/buildroot:blue

  # and auth options, registries, secrets, etc

  [prod-osd]
  registry_api_versions = v1,v2
  openshift_url = https://api.prod-example.openshift.com/
  node_selector.x86_64 = none
  use_auth = true
  token_file =
    /var/run/secrets/atomic-reactor/workertoken/osd-serviceaccount-token
  # and auth options, registries, secrets, etc

In this configuration file there are two worker clusters, one which
builds for both x86_64 and ppc64le platforms using nodes with specific
labels (prod-mixed), and another which only accepts x86_64 builds
(prod-osd).

Note that although "registry_api_versions" lists v1, ppc64le builds
will not publish v1 images as there is no "platform:ppc64le" section
containing "enable_v1 = true".

Pushing built images to Pulp
----------------------------

