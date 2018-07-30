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

enable_v1 (optional)
  if support for the Docker Registry HTTP API v1 (pulp_push etc)
  may be included for this platform, the value should be "true"; the
  default is "false"

When creating a worker build for an OSBS instance, both the
"registry_api_versions" key for the instance and the "enable_v1" key
for the platform will be consulted. They must both instruct v1 support
to enable publishing v1 images. If either does not instruct v1 support,
v1 images will not be published.

At most one platform may have "enable_v1 = true".

For example::

  [platform:x86_64]
  architecture = amd64
  enable_v1 = true

  [platform:ppc]
  architecture = ppc64le

  [instance1]
  registry_api_versions = v1,v2
  ...

  [instance2]
  registry_api_versions = v2
  ...

In the above configuration, worker builds created using instance1 for
the x86_64 platform will publish v1 images as well as v2 images. Other
platforms on instance1, and all platforms on instance2, will only
publish v2 images.

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
  List of signing intents in their restrictive order.

default_signing_intent
  Name of the default signing intent to be used when one is not provided
  in ``container.yaml``.


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

Priority of Container Image Builds
----------------------------------

For a build system it's desirable to prioritize different kinds of builds in
order to better utilize resources. Unfortunately, OpenShift's scheduling
algorithm does not support setting a priority value for a given build. To
achieve some sort of build prioritization, we can leverage node selectors to
allocate different resources to different build types.

Consider the following types of container builds builds:

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

Providing the osbs-client logs in the Koji task should give users a better
understanding of any delays due to scheduling.
