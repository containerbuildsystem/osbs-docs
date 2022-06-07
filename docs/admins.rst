Administration and Maintenance
==============================

Understanding build parameters
------------------------------

Please refer to :doc:`Build Parameters <build_parameters>` for
information on how options are configured within OSBS builds.


.. _configuring-osbs-client:

Configuring osbs-client
-----------------------

reactor_config_map
~~~~~~~~~~~~~~~~~~~~~

``reactor_config_map`` specifies the name of a
Kubernetes configmap holding :ref:`config.yaml`. A pre-build plugin will
read its value from REACTOR_CONFIG environment variable.


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

Specifies the build image to be used for building container
images. This is a full reference to
a specific container image in a container registry (including
registry, repository, and either tag or digest).

Updating this globally effectively deploys a different version of
OSBS.

cleanup_used_resources
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Specifies the cleanup strategy for OSBS builds. By default this is set to
``true``. When it is set to ``true`` it'll remove the build pipeline on a
completed or failed build.

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

OSBS builds container images using podman-remote on remote VMs (hosts).
OSBS needs only ssh key and hostname to connect to remote host and
execute podman.

Example of reactor-config-map runtime configuration::

  ---
  remote_hosts:
    slots_dir: path/foo
    pools:
      x86_64:
        osbs-remote-hosts-1-x86_64:
          enabled: true
          auth: /secret-path
          username: podman-user
          slots: 1
          socket_path: /run/user/2022/podman/podman.sock
        osbs-remote-hosts-2-x86_64:
          enabled: false
          auth: /secret-path
          username: podman-user
          slots: 2
          socket_path: /run/user/2022/podman/podman.sock
      ppc64le:
        osbs-remote-hosts-1-ppc64le:
          enabled: true
          auth: /secret-path
          username: podman-user
          slots: 3
          socket_path: /run/user/2022/podman/podman.sock

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

remote_hosts
''''''''''''

This section provides a shared path to the slots directory.
The slots directory holds files with information about ongoing
builds.

The section also provides Remote host pools objects of specific platforms.
Each platform object contains hosts with the same architecture.

Host object provides key information for building images:
- hosts with the enabled key set to false are ignored

- `auth` provides file path to SSH key

- `slots` represent maximum host capacity. The number of builds which
  can be built in parallel

- the host for building images will be picked based on current
  availability defined by a ratio of `available slots` divided by `all
  slots`

- the remote host build is submitted to whichever host has the lowest
  load; in this way, even load distribution across all hosts is
  enforced

This mechanism can also be used to temporarily disable a remote host by
removing it from the list or adding ``enabled: false`` to
the host description for each platform.

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

Limiting image size
'''''''''''''''''''

You can check the binary image's size before it is pushed to a registry. If it
exceeds the configured size, the built image will not be pushed and the build
fails.

A typical configuration in reactor config map looks like::

  image_size_limit:
    binary_image: 10000

The value is the size in bytes of uncompressed layers. When either
``binary_image`` or ``image_size_limit`` is omitted, or if ``binary_image`` is
set to ``0``, the check will be skipped.

Custom CA bundle
''''''''''''''''

It is allowed to specify a custom CA bundle explicitly to include self-signed
certificates. If set, it will be injected into every YUM repository added by
users. The custom CA bundle is used during the container build process only.

Set the CA bundle certificate by config ``builder_ca_bundle`` at the top level
of the reactor config. The value must be a file name with an absolute path to
an existing certificate file inside the builder image. For example, if the
required self-signed certificate is included in the file
``/etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem``, then the config is:

.. code-block:: yaml

   builder_ca_bundle: /etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem

Setting up koji for container image builds
------------------------------------------

Example configuration file: Koji builder
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Configuration of the ``osbs.conf`` used by the Koji builder is required for
binary and source builds, and each of build type has it's own section.
The minimal configuration for binary and source build would include::

  [general]

  # default configuration section for source builds
  [default_source]
  openshift_url = https://source.example.com:8443/
  # openshift namespace
  namespace = source_example
  use_auth = true
  verify_ssl = true
  # path to source pipeline run
  pipeline_run_path = /usr/share/osbs/source-container-pipeline-run.yaml
  # name of config map for regular builds
  reactor_config_map = reactor-config-map
  # name of config map for scratch builds
  reactor_config_map_scratch = reactor-config-map-scratch
  # path to openshift token
  token_file = /etc/osbs/openshift-serviceaccount.token
  # also possible to specify directly token with:
  # token = ...

  # default configuration section for binary builds
  [default_binary]
  openshift_url = https://binary.example.com:8443/
  # openshift namespace
  namespace = binary_example
  use_auth = true
  verify_ssl = true
  # path to binary pipeline run
  pipeline_run_path = /usr/share/osbs/binary-container-pipeline-run.yaml
  # name of config map for regular builds
  reactor_config_map = reactor-config-map
  # name of config map for scratch builds
  reactor_config_map_scratch = reactor-config-map-scratch
  # path to openshift token
  token_file = /etc/osbs/openshift-serviceaccount.token
  # also possible to specify directly token with:
  # token = ...



Pipeline run template
'''''''''''''''''''''
Osbs-client requires path to tekton PipelineRun template (``pipeline_run_path``)
which is used to create PipelineRun tekton object. PipelineRun template must be
created according to OSBS tekton Pipeline definitions with modifications suitable for
the deployment (different way of getting PVC, extra labels, extra tekton configuration,
etc..).


Example:

.. code-block:: yaml

  ---
  apiVersion: tekton.dev/v1beta1
  kind: PipelineRun
  metadata:
    name: '$osbs_pipeline_run_name'
  spec:
    pipelineRef:
      name: source-container-0-1
    params:
      - name: OSBS_IMAGE
        value: "registry/tests_image:latest"
      - name: USER_PARAMS
        value: '$osbs_user_params_json'  # json must be in ' '
    workspaces:
      - name: ws-context-dir
        volumeClaimTemplate:
          metadata:
            name: source-container-context-pvc
            namespace: '$osbs_namespace'
            annotations:
              kubernetes.io/reclaimPolicy: Delete
          spec:
            accessModes:
              - ReadWriteOnce
            resources:
              requests:
                storage: 100Mi
      - name: ws-build-dir
        volumeClaimTemplate: "just example, any other method"
      - name: ws-registries-secret
        secret:
          secretName: registries-secret
      - name: ws-koji-secret
        secret:
          secretName: koji-secret
      - name: ws-reactor-config-map
        configmap:
          name: '$osbs_configmap_name'
    timeout: 3h

Osbs-client provides extra template variables, starting with prefix ``$osbs_``
to inject OSBS specific data.

.. list-table:: Substitution variables
   :header-rows: 1

   * - Variable
     - Description
   * - $osbs_configmap_name
     - Name of configmap to be used in build (taken from osbs-client config)
   * - $osbs_namespace
     - OSBS namespace used for build (taken from osbs-client config)
   * - $osbs_pipeline_run_name
     - Pipeline run name created by OSBS (required)
   * - $osbs_user_params_json
     - OSBS user parameters encoded in JSON. It's JSON string; don't forget to use it as `'$osbs_user_params_json'`


Including OpenShift build annotations in Koji task output
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Successful container image builds may include a ``build_annotations.json`` file
in the task output. This file includes a subset of the OpenShift annotations
for the container build triggered by the Koji task in question.

The ``koji-containerbuild`` builder plugin hardcodes the list of annotations to
include in the generated file. If none of the predefined annotations are present
and ``build_annotations.json`` would thus be empty, the file is omitted from the
task output entirely.

The ``build_annotations.json`` file is a JSON object with first level key/values
where each key is an OpenShift build annotation mapped to it's value.

Note that, confusingly, the annotation values in ``build_annotations.json``
do not in fact come from annotations. Due to seemingly unreliable behavior of
updating annotations on Tekton PipelineRun objects, ``koji-containerbuild``
takes the values from Tekton results instead. OSBS pipelines provide only the
required subset of annotations via Tekton results.


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
    skip_all_allow_list:
      - koji_package1
      - koji_package2

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

skip_all_allow_list
  List of koji packages which are allowed to use ``skip_all`` option in
  the ``operator_manifests`` section of ``container.yaml``.

.. _operator-csv-modifications-admin:

Enabling operator CSV modifications
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To allow operator CSV modifications attributes which are allowed to be updated
must be added to the ``allowed_attributes`` list.

Example:

.. code-block:: yaml

  operator_manifests:
    csv_modifications:
      allowed_attributes:
      -  ["spec", "skips",]
      -  ["spec", "version",]
      -  ["metadata", "substitutes-for",]
      -  ["metadata", "name",]

csv_modifications
  Section with configuration related to operator CSV modifications (for future expansion)

allowed_attributes
  List of paths to attributes (defined as list of strings) which are allowed to be modified


.. _package_mapping.json: https://github.com/containerbuildsystem/atomic-reactor/blob/master/atomic_reactor/schemas/package_mapping.json

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

.. _allow-multiple-remote-sources:

Allowing multiple remote sources
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
To enable support for multiple remote sources, set
the ``allow_multiple_remote_sources`` flag to ``true`` in
``reactor_config_map``.

.. code-block:: yaml

    allow_multiple_remote_sources: true



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

.. _cachito: https://github.com/release-engineering/cachito


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

.. _`config.json`: https://github.com/containerbuildsystem/atomic-reactor/blob/master/atomic_reactor/schemas/config.json
