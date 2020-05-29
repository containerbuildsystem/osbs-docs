Administration and Maintenance
==============================

Understanding build parameters
------------------------------

Please refer to :doc:`Build Parameters <build_parameters>` for
information on how options are configured within OSBS builds.


.. _configuring-osbs-client:

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

reactor_config_map
~~~~~~~~~~~~~~~~~~~~~

``reactor_config_map`` specifies the name of a
Kubernetes configmap holding :ref:`config.yaml`. A pre-build plugin will
read its value from REACTOR_CONFIG environment variable.

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

Node selector
~~~~~~~~~~~~~

When an entry with the pattern ``node_selector.platform`` (for some
*platform*) is specified, builds for this platform submitted to this
cluster must include the given node selector, so as to run on a node
of the correct architecture. This allows for installations that have
mixed-architecture clusters and where node labels differentiate
architecture.

If the value is ``none``, this platform is the only one available and
no node selector is required.

Platform description
~~~~~~~~~~~~~~~~~~~~

When a section name begins with "platform:" it is interpreted not as
an OSBS instance but as a platform description. The remainder of the
section name is the platform name being described. The section has the
following keys:

architecture (optional)
  the GOARCH for the platform -- the platform name is assumed to be
  the same as the GOARCH if this is not specified

build_from
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Specifies the build image (AKA "buildroot") to be used for building container
images, to be set in the ``Build``/``BuildConfig`` OpenShift objects under the
``.spec.strategy.customStrategy.from`` object. This can be a full reference to
a specific container image in a container registry; or it can reference an
ImageStreamTag object.

Updating this globally effectively deploys a different version of
OSBS.

It takes one of the following forms:

imagestream:*imagestream*:*tag*
  use the image from the specified OpenShift ImageStreamTag

image:*pullspec*
  pull the image from the specified pullspec (including
  registry, repository, and either tag or digest)

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
order of preference. It may also contain additional environment configuration
such as ODCS integration.

The runtime configuration will take the form of a Kubernetes secret
with content as in the example below::

  ---
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

  odcs:
    signing_intents:
    - name: release
      keys: [AB123]
    - name: beta
      keys: [BT456, AB123]
    - name: unsigned
      keys: []
    # Value must match one of the names above.
    default_signing_intent: release


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

.. _config.yaml-odcs:

odcs
''''

Section used for ODCS related configuration.

signing_intents
  List of signing intents in their restrictive order. Since composes can be
  renewed in ODCS, OSBS needs to check if the signing keys used in a compose to
  be renewed are still valid. If the signing keys are not valid anymore, i.e.,
  keys were removed from the OSBS signing intent definition, OSBS will request
  ODCS to update the compose signing keys. For OSBS to identify the proper
  signing intent in such cases, you should not remove signing keys from signing
  intents. Instead, move the keys that should not be valid anymore from the
  ``keys`` map to the ``deprecated_keys`` map in the relevant signing intent
  definitions. Failing to do so will result in build failures when renewing
  composes with old signing intent key sets.

default_signing_intent
  Name of the default signing intent to be used when one is not provided
  in ``container.yaml``.

.. _config.yaml-build_env_vars:

build_env_vars
''''''''''''''

Define variables that should be propagated to the build environment here.
Note that some variables are reserved and defining them will cause an error,
e.g. ``USER_PARAMS``, ``REACTOR_CONFIG``.

For example, you might want to set up an HTTP proxy:

.. code-block:: yaml

  build_env_vars:
  - name: HTTP_PROXY
    value: "http://proxy.example.com"
  - name: HTTPS_PROXY
    value: "https://proxy.example.com"
  - name: NO_PROXY
    value: localhost,127.0.0.1


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

  [default]
  openshift_url = https://orchestrator.example.com:8443/
  build_image = example.registry.com/buildroot:blue

  distribution_scope = public

  can_orchestrate = true  # allow orchestrator builds

  # This secret contains configuration relating to which worker
  # clusters to use and what their capacities are:
  reactor_config_map = reactorconf

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

  reactor_config_map = reactorconf
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

  [prod-mixed]
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

.. _whitelist-annotations:

Including OpenShift build annotations in Koji task output
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

It is possible to include a ``build_annotations.json`` file in the task output
of successful container image builds. This file may include any wanted
OpenShift build annotations for the container build triggered by the Koji task
in question.

The ``koji-containerbuild`` plugin looks for a
``koji_task_annotations_whitelist`` annotation in the OpenShift build
annotations. This key should hold a list of annotations to be whitelisted for
inclusion in the ``build_annotations.json`` file.

If an empty ``build_annotations.json`` file would be generated through the
process described above, the file is omitted from the task output. For
instance, ``koji_task_annotations_whitelist`` could be empty, or the
whitelisted annotations not present in OpenShift build annotations.

To whitelist the desired annotations in the ``koji_task_annotations_whitelist``
OpenShift annotation described above, you can use the
``task_annotations_whitelist`` ``koji`` configuration in the
``reactor_config_map``. See :ref:`config.yaml` for further reference.

The ``build_annotations.json`` file is a JSON object with first level
key/values where each key is a whitelisted OpenShift build annotation mapped to
it's value.

Priority of Container Image Builds
----------------------------------

For a build system it's desirable to prioritize different kinds of builds in
order to better utilize resources. Unfortunately, OpenShift's scheduling
algorithm does not support setting a priority value for a given build. To
achieve some sort of build prioritization, we can leverage node selectors to
allocate different resources to different build types.

Consider the following types of container builds:

- *scratch build*
- *explicit build*
- *auto rebuild*

As the name implies, *scratch builds* are meant to be used as a one-off
unofficial container build. No guarantees are made for storing the created
container images long term. It’s also not meant to be shipped to customers.
These are clearly low priority builds.

*Explicit builds* are those triggered by a user, either directly via fedpkg/koji
CLI, or indirectly via pungi (as in the case of base images). These are official
builds that will go through the normal life cycle of being tested and,
eventually, shipped.

*Auto rebuilds* are created by OpenShift when a change in the parent image is
detected. It’s likely that layered images should be rebuilt in order to pick up
changes in latest parent image.

For any *explicit build* or *auto rebuild*, they may or may not be high
priority. In some cases, a build is high priority due to a security fix, for
instance. In other cases, it could be due to an in-progress feature. For this
reason, it cannot be said that all *explicit builds* are higher priority than
*auto rebuilds*, or vice-versa.

However, *auto rebuilds* have the potential of completely consuming OSBS’s
infrastructure. There must be some mechanism to throttle the amount of *auto
rebuilds*. For this reason, OSBS uses a different node selector for each
different build type:

- *scratch build*: builds_scratch=true
- *explicit build*: builds_explicit=true
- *auto rebuild*: builds_auto=true

By controlling each type of builds individually, OSBS will have the necessary
control for adjusting its infrastructure.

For example, consider an OpenShift cluster with 5 compute nodes:

======  =================== ==================== ================
Node    builds_scratch=true builds_explicit=true builds_auto=true
======  =================== ==================== ================
Node 1  ✔                   ✔                    ✔
Node 2  ✗                   ✔                    ✗
Node 3  ✗                   ✗                    ✔
Node 4  ✗                   ✔                    ✔
Node 5  ✗                   ✔                    ✔
======  =================== ==================== ================

In this case, *scratch builds* can be scheduled only on **Node 1**; *explicit
builds* on any node except **Node 3**; and auto builds on any node except **Node
2**.

Worker Builds Node Selectors
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The build type node selectors are only applied to worker builds. This gives
more granular control over available resources. Since worker builds are the ones
that actually perform the container image building steps, it requires more
resources than orchestrator builds. For this reason, a deployment is more likely
to have more nodes available for worker builds than orchestrator builds. This is
important because the amount of nodes available defines the granularity of how
builds are spread across the cluster.

For instance, consider a large deployment in which only 2 orchestrator nodes are
needed.  If build type node selectors are applied to orchestrator builds, builds
can only be throttled by a factor of 2. In contrast, this same deployment may
use 20 worker builds, allowing builds to be throttled by a factor of 20.

Orchestrator Builds Allocation
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Usually in a deployment, the amount of allowed orchestrator builds matches the
amount of allowed worker builds for any given platform. Additional orchestrator
builds should be allowed to fully leverage the build type node selectors on
worker builds since some orchestrator builds will wait longer than usual for
their worker builds to be scheduled. This provides a buffer that allows
OpenShift to properly schedule worker builds according to their build type via
node selectors. Because OpenShift scheduling is used, worker builds of same type
will run in the order they were submitted.


Koji Builder Capacity
~~~~~~~~~~~~~~~~~~~~~

The task load of the Koji builders used by OSBS will not reflect the actual load
on the OpenShift cluster used by OSBS. The disparity is due to auto rebuilds not
having a corresponding Koji task. This creates a scenario where a buildContainer
Koji task is started, but the OpenShift build remains in pending state. The Koji
builder capacity should be set based on how many nodes allow **scratch builds**
and/or **explicit builds**. In the example above, there are 4 nodes that allow
such builds.

The log file, *osbs-client.log*, in a Koji task gives users a better
understanding of any delays due to scheduling.

Operator manifests
------------------

Supporting Operator Manifests extraction
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To support the operator_ manifests extraction, as described in
:ref:`Operator manifests <operator-manifests>`, the `operator-manifests`
BType must be created in koji. This is done by running

.. code-block:: shell

  koji call addBType operator-manifests

.. _operator: https://coreos.com/operators/

Enabling Operator Manifests digest pinning (and other replacements)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To enable digest pinning and other replacements of image pullspecs for
:ref:`operator manifest bundle <operator-bundle>` builds, atomic-reactor
config must include the ``operator_manifests`` section. See configuration
details in `config.json`_.

Example:

.. code-block:: yaml

  operator_manifests:
    allowed_registries:
      - private-registry.example.com
      - public-registry.io
    repo_replacements:
      - registry: private-registry.example.com
        package_mappings_url: https://somewhere.net/package_mapping.yaml
    registry_post_replace:
      - old: private-registry.example.com
        new: public-registry.io

allowed_registries
  List of allowed registries for images *before* replacement. If any image is
  found whose registry is not in ``allowed_registries``, build will fail. This
  key is required.

  Should be a subset of ``source_registry + pull_registries`` (see
  `config.json`_).

repo_replacements
  Each registry may optionally have a "package mapping" - a YAML file that
  contains a mapping of [package name => list of repos] (see
  `package_mapping.json`_). The file needs to be uploaded somewhere that OSBS
  can access, and will be downloaded from there during build if necessary.

  Images from registries with a package mapping will have their namespace/repo
  replaced. OSBS will query the registry to find the package name for the image
  (determined by the component label) and get the matching replacement from the
  mapping file. If there is no replacement, or if there is more than one, build
  will fail and user will have to specify one in ``container.yaml``.

registry_post_replace
  Each registry may optionally have a replacement. After pinning digest and
  replacing namespace/repo, all ``old`` registries in image pullspecs will be
  replaced by their ``new`` replacements.

.. _package_mapping.json: https://github.com/containerbuildsystem/atomic-reactor/blob/master/atomic_reactor/schemas/package_mapping.json

.. _omps-integration:

Enabling integration with OMPS service
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To enable optional integration with OMPS_ service to allow automatically pushing
operators manifests to application registry (like quay_) ``omps`` configuration
section must be added into atomic-reactor configuration.
See configuration details in `config.json`_.

Example:

.. code-block:: yaml

  omps:
    omps_url: https://omps-service.example.com
    omps_namespace: organization
    omps_secret: /dir/where/token/file/will/be/mounted
    appregistry_url: https://quay.io/cnr

.. _OMPS: https://github.com/release-engineering/operators-manifests-push-service
.. _quay: https://quay.io/application/
.. _`config.json`: https://github.com/containerbuildsystem/atomic-reactor/blob/master/atomic_reactor/schemas/config.json

.. _cachito-integration:

Cachito integration
-------------------

cachito_ caches specific versions of upstream projects source code along with
dependencies and provides a single tarball with such content for download upon
request. This is important when you want track the version of a project and its
dependencies in a more robust manner, without handing control of storing and
handling the source code for a third party (e.g., if tracking is performed in
an external git forge, someone could force push a change to the repository or
simply delete it).

OSBS is able to use cachito to handle the source code used to build a container
image. The source code archive provided by cachito and the data used to perform
the cachito request may then be attached to the koji build output, making it
easier to track the components built in a given container image.

This section describes how to configure OSBS to use cachito as described above.
:ref:`cachito-usage` describes how to get OSBS to use cachito in
a specific container build, as an OSBS user.

Configuring your cachito instance
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To enable cachito integration in OSBS, you must use the ``cachito``
configuration in the ``reactor_config_map``. See configuration details in
`config.json`_.

Example:

.. code-block:: yaml

  cachito:
    api_url: https://cachito.example.com
    auth:
      ssl_certs_dir: /dir/with/cert/file

Configuring koji
~~~~~~~~~~~~~~~~

Adding remote-sources BType
''''''''''''''''''''''''''''

To fully support cachito_ integration, as described in
:ref:`cachito-integration`, the `remote-sources`
BType must be created in koji. This is done by running

.. code-block:: shell

  koji call addBType remote-sources

This new build type will hold cachito related build artifacts generated in
atomic-reactor, which should include a tarball with the upstream source code
for the software installed in the container image and a `remote-source.json`
file, which is a JSON representation of the source request sent to cachito by
atomic-reactor. This JSON file includes information such as the repository from
where cachito downloaded the source code and the revision reference that was
downloaded (e.g., a git commit hash).

Whitelisting `remote_source_url` build annotation
'''''''''''''''''''''''''''''''''''''''''''''''''
In addition to adding the new BType to koji, you may also want to whitelist the
OpenShift `remote_source_url` build annotation. This is specially useful for
scratch builds, where a koji build is not generated and users would not have
information about how the sources were fetch for that build easily available.
whitelist-annotations_ describes the steps needed to whitelist OpenShift build
annotations.

.. _cachito: https://github.com/release-engineering/cachito

Troubleshooting
---------------

Builds will automatically cancel themselves if any worker takes more than 3
hours to complete or the entire task takes more than 4 hours to complete.
Administrators can override these run time values with the
``worker_max_run_hours`` and ``orchestrator_max_run_hours`` settings in the
``osbs.conf`` file.

Obtaining Atomic Reactor stack trace
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

atomic-reactor captures *SIGUSR1* signals. When receiving such signal,
atomic-reactor responds by showing the current stack trace for every thread it
was running when the signal was received.

An administrator can use this to inspect the orchestrator or a specific
worker build. It is specially useful to diagnose stuck builds.

As an administrator, use ``podman kill --signal=SIGUSR1
<BUILDROOT_CONTAINER>`` or ``podman exec <BUILDROOT_CONTAINER> kill -s SIGUSR1
1`` to send the signal to the buildroot container you wish to inspect.
atomic-reactor will dump stack traces for all its threads into the buildroot
container logs. For instance::

    Thread 0x7f6e88a1b700 (most recent call first):
      File "/usr/lib/python2.7/site-packages/atomic_reactor/inner.py", line 277, in run
      File "/usr/lib64/python2.7/threading.py", line 812, in __bootstrap_inner
      File "/usr/lib64/python2.7/threading.py", line 785, in __bootstrap

    Current thread 0x7f6e95dbf740 (most recent call first):
      File "/usr/lib/python2.7/site-packages/atomic_reactor/util.py", line 74, in dump_traceback
      File "/usr/lib/python2.7/site-packages/atomic_reactor/util.py", line 1562, in dump_stacktraces
      File "/usr/lib64/python2.7/socket.py", line 476, in readline
      File "/usr/lib64/python2.7/httplib.py", line 620, in _read_chunked
      File "/usr/lib64/python2.7/httplib.py", line 578, in read
      File "/usr/lib/python2.7/site-packages/urllib3/response.py", line 203, in read
      File "/usr/lib/python2.7/site-packages/docker/client.py", line 247, in _stream_helper
      File "/usr/lib/python2.7/site-packages/atomic_reactor/util.py", line 297, in wait_for_command
      File "/usr/lib/python2.7/site-packages/atomic_reactor/plugins/build_docker_api.py", line 46, in run
      File "/usr/lib/python2.7/site-packages/atomic_reactor/plugin.py", line 239, in run
      File "/usr/lib/python2.7/site-packages/atomic_reactor/plugin.py", line 449, in run
      File "/usr/lib/python2.7/site-packages/atomic_reactor/inner.py", line 444, in build_docker_image
      File "/usr/lib/python2.7/site-packages/atomic_reactor/inner.py", line 547, in build_inside
      File "/usr/lib/python2.7/site-packages/atomic_reactor/cli/main.py", line 95, in cli_inside_build
      File "/usr/lib/python2.7/site-packages/atomic_reactor/cli/main.py", line 292, in run
      File "/usr/lib/python2.7/site-packages/atomic_reactor/cli/main.py", line 310, in run
      File "/usr/bin/atomic-reactor", line 11, in <module>

In this example, this build is stuck talking to the docker client (``docker/client.py``).
