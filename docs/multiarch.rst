Multi-platform container image builds
=====================================

This is the "future state" description of how container images will be
built for multiple platforms.

A single git commit will be used as the source for building multiple
container images, each suitable for a different platform (CPU
architecture, variant, required CPU features, etc).

To support multi-arch builds, two types of OpenShift Build will be
used:

- *worker builds* which will perform the "docker build" step as
  before (on compute nodes of the appropriate architecture)
- *orchestrator build* which will create and manage the worker builds,
  and group the resulting image manifests into a `manifest list`_

.. _`manifest list`: https://docs.docker.com/registry/spec/manifest-v2-2/#manifest-list

This will be a different "arrangement" (see :ref:`build process`) of
plugins across the orchestrator and worker builds.

In essence, only compute nodes in the required architectures are needed to
perform the container-building step. However, OpenShift does not support nodes
of different architectures in the same cluster. The recommended approach is to
use a different cluster for each architecture.

The orchestrator build will make use of :ref:`config.yaml` to discover
which worker clusters to direct builds to and
:ref:`client_config_secret` to discover how to connect to those
clusters, including whether/which `node selector`_ is required for
each platform.

.. _`node selector`: https://docs.openshift.org/latest/admin_guide/managing_projects.html#developer-specified-node-selectors

Orchestration arrangement for multi-platform builds
---------------------------------------------------

When a new multi-platform build is required, one designated cluster
will always be asked to perform it. This cluster will run
atomic-reactor in "orchestration" mode. As part of its operation it
will in turn ask worker clusters to perform platform-specific builds,
which each run atomic-reactor in "worker" mode.

Steps performed in the orchestrator build are:

- For base images, create the base image filesystems for all required
  architectures using Koji

- For layered images:

  * inspect the parent image's environment variables (this is to allow
    substitution to be performed in the "release" label)

- Create worker builds on other clusters (or this cluster) as needed;
  if any worker build fails the orchestrator build must also fail (but
  other worker builds should be left to continue); if the orchestrator
  build is cancelled, worker builds will be cancelled. See `Worker
  build`_ for more details about how these builds behave.

- Collect package content from all builds and compare to ensure common
  components are exactly matching (for RPMs: version-release and
  keys used for signatures)

- Create a manifest list in the container registry, holding image
  manifest digests from all the worker builds

- If Pulp integration is enabled:

  * On success, sync and publish new Pulp content from this build

  * On failure, remove the Pulp content

- If Koji integration is enabled:

  - Combine Koji metadata fragments from temporary storage (see
    :ref:`Metadata Fragment Storage`)

  - Create a Koji Build. Note: no need to upload image tar archives as
    worker builds have done this

- Update this OpenShift Build with annotations about output,
  performance, errors, worker builds used, etc

- Perform any clean-up required:

  * If Pulp integration is enabled, delete the manifest list from the
    registry

Worker build
~~~~~~~~~~~~

The orchestration step will create OpenShift builds for performing the
worker builds. Inside each of these platform-specific OpenShift
builds, atomic-reactor will execute these steps as plugins:

- For layered images, pull the parent image and note its image ID

- Make alterations to the Dockerfile, including setting the
  ``architecture`` label appropriately

- Build the container

- Squash the image

- Compress the docker image tar archive

- Upload this compressed tar archive to Pulp (if Pulp integration and
  v1 support are both enabled)

- Tag and push the image to the container registry (see :ref:`image-tags`)

- Query the image to discover installed RPM packages (by running
  ``rpm`` inside it)

- If Koji integration is enabled:

  - Upload image tar archive to Koji but do not create a Koji Build

  - Upload the Koji metadata fragment (buildroot information, built
    image information) to temporary storage (see :ref:`Metadata
    Fragment Storage`)

- Update this OpenShift Build with annotations about output,
  performance, errors, etc

.. graphviz:: images/multi-during-build.dot
   :caption: During build

.. graphviz:: images/multi-after-build.dot
   :caption: After build

Submitting builds
-----------------

A new optional parameter ``--arches`` will be added to the
``container-image`` subcommand provided by pyrpkg. If ``--arches`` is
supplied, ``--scratch`` must also be supplied. It will pass the
parameter ``arches`` to the Koji task (implemented by the
``koji-containerbuild`` plugin for Koji). This mirrors how the
equivalent parameter works when building RPMs.

When supplied for a scratch build this parameter overrides the default
set of architectures to build for, which comes from the Koji build
target. If an image cannot be built for any supplied architectures the
build will fail.

Building base images
--------------------

The atomic-reactor ``add_filesystem`` plugin is responsible for
creating a Koji image-build task and streaming the output of that task
into the initial container image layer. It does this with the aid of
an ``image-build.conf`` file in the git repository.

For multi-platform builds the Koji image-build task needs to be
started by the orchestrator build and configured to build for multiple
architectures. This Koji task will have multiple output files, one for
each architecture. The ``image-build.conf`` file in the git
repository should be changed so that it no longer specifies any
architecture, as atomic-reactor will supply this field.

Having the orchestrator build do this step, which mostly involves
waiting for the Koji task to finish, results in better (more accurate)
resource allocation. Orchestrator builds will have slimmer resource
requests than those of worker builds.

After the Koji task has finished, the worker builds then need to be
instructed to take their input from a specific output of that
task. The ``add_filesystem`` plugin will need changes for this:

- it will need a parameter to tell it to create a multi-platform
  image-build task and not stream the output of that task. This
  parameter will be set for the orchestrator build.

- it can already be told to take its input from the output of a
  specific Koji task, but will need to be able to decide which
  particular task output file is required by parsing the output
  filenames and looking for the platform name. This parameter will be
  set for the worker build.

Scratch builds
--------------

There are no changes to how scratch builds are performed, only some
parts of the implementation will move around. Some build steps will be
omitted when performing scratch builds:

- the resulting manifest list will only be tagged using the unique tag
  including the timestamp
- the result will not be imported into Koji in the orchestrator build

Chain rebuilds
--------------

OpenShift Build Triggers, and atomic-reactor plugins dealing with
ImageStreams or triggers, are only applicable to the orchestrator
BuildConfigs. The x86_64 image stream tags (from Pulp's crane, when
Pulp integration is enabled) will be used for triggering builds, and
Pulp repositories will be published by the orchestrator build, not the
worker builds.

Although worker builds will be associated with BuildConfigs for
convenience of grouping historical builds for the same component in
the "console" interface, no worker BuildConfigs will have triggers.

Low priority builds
-------------------

For scratch builds and for triggered rebuilds, node selectors will be
used to restrict the set of nodes which may perform these low-priority
builds. The node selector for doing this will be combined with the
node selector for selecting platform-specific nodes.

Cancellation and failure
------------------------

When a build is canceled in Koji this should be correctly propagated
all the way down to the worker builds:

- koji_containerbuild calls the osbs-client API method to cancel
  the (orchestration) build
- osbs-client calls the OpenShift API method to cancel the
  orchestrator build in OpenShift
- OpenShift sends a signal to atomic-reactor
- atomic-reactor handles this signal by calling the osbs-client API
  method to cancel each worker build
- Each osbs-client invocation calls the OpenShift API method to cancel
  a worker builder
- Each worker instance of atomic-reactor handles the signal it gets
  sent by running exit plugins, which perform clean-up operations
- The orchestrator instance of atomic-reactor finishes by running its
  exit plugins

In the case of a build for one platform failing, builds for other
platforms will continue. Once all have either succeeded or failed, the
orchestrator build will fail. No content will be available from the
registry.

Client Configuration
--------------------

The osbs-client configuration file format will be augmented with
instance-specific field ``node_selector``, and new sections for
describing platforms.

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

Implementation of this requires a new optional parameter platform for
the API method ``create_prod_build`` specifying which platform a build
is required for. If no platform is specified, no node selector will be
used.

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

  # This node selector will be applied to triggered rebuilds:
  low_priority_node_selector = lowpriority=true

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

  low_priority_node_selector = lowpriority=true
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

Also shown is the configuration for `Scratch builds`_, which will be
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

  # This node selector, combined with the platform-specific node
  # selector, will be applied to worker builds.
  low_priority_node_selector = lowpriority=true

  # and auth options, registries, secrets, etc

  [prod-osd]
  registry_api_versions = v1,v2
  openshift_url = https://api.prod-example.openshift.com/
  node_selector.x86_64 = none
  use_auth = true
  token_file =
    /var/run/secrets/atomic-reactor/workertoken/osd-serviceaccount-token
  low_priority_node_selector = lowpriority=true
  # and auth options, registries, secrets, etc

In this configuration file there are two worker clusters, one which
builds for both x86_64 and ppc64le platforms using nodes with specific
labels (prod-mixed), and another which only accepts x86_64 builds
(prod-osd).

Note that although "registry_api_versions" lists v1, ppc64le builds
will not publish v1 images as there is no "platform:ppc64le" section
containing "enable_v1 = true".

Client API changes
------------------

get_orchestrator_build_logs
~~~~~~~~~~~~~~~~~~~~~~~~~~~

This new API method will take the following parameters:

build_id (str)
  name of the orchestrator build

follow (bool, defaults to False)
  whether to stream logs

wait_if_missing (bool, defaults to False)
  whether to wait for the build to exist first

It will call ``get_build_logs(decode=True)`` and **yield** a named
tuple with fields 'platform' and 'line'.

platform (str)
  platform name if worker build, else None

line (str)
  log line

See :ref:`logging` for more details.

create_worker_build
~~~~~~~~~~~~~~~~~~~

This existing API method will gain additional optional parameters:

filesystem_koji_task_id
  Koji Task ID of image-build task. This will be supplied as a
  "from_task_id" argument to the add_filesystem plugin in the worker
  build.

koji_upload_dir
  Relative path to use when uploading files to Koji. This will be
  supplied as a "koji_upload_dir" argument to the koji_upload plugin
  in the worker build.


create_config_map
~~~~~~~~~~~~~~~~~

This new API method will be used by a worker build to create a
ConfigMap object in which the metadata fragment for the image build
will be stored. It takes two parameters:

name
  This string is the name of the ConfigMap object to create

data
  This is a dict whose keys and values should be stored in the
  ConfigMap. For :ref:`Metadata Fragment Storage` it is expected that the
  value will be a JSON string.

get_config_map
~~~~~~~~~~~~~~

This new API method will be used by `fetch_worker_metadata`_. It takes
a single parameter.

name
  This string is the name of the ConfigMap object to retrieve

It should return a ConfigMapResponse object (this is a new object
similar to e.g. PodResponse) which allows access to keys and values
within the ConfigMap.

delete_config_map
~~~~~~~~~~~~~~~~~

This new API method will be used by the orchestrator build to remove
ConfigMap objects created by the worker builds. It takes a single
parameter.

name
  This string is the name of the ConfigMap object to delete

ConfigMapResponse
~~~~~~~~~~~~~~~~~

This object is similar to e.g. PodResponse, but encapsulates the
response to a a request to get a ConfigMap. It should provide access
to keys and values using a method such as:

get_items
  This method takes no parameters and returns a dict whose keys are
  the keys within the ConfigMap, and whose values are the
  corresponding values for those ConfigMap keys.

Anatomy of an orchestrator build
--------------------------------

When creating an OpenShift build to run atomic-reactor in
"orchestration" mode, the "build" step will be chosen to be the plugin
which performs orchestration rather than the plugin which simply runs
"docker build".

The configuration for this plugin will include the osbs-client
instance configuration for the named workers in addition to the list
of platforms to build for.

The purpose of the orchestrator build is to choose a worker cluster,
create a worker build in it, monitor worker builds, and group them
into a manifest list. Below is an example of the
ATOMIC_REACTOR_PLUGINS environment variable for an orchestrator build.

::

   {
    "prebuild_plugins": [
      {
        "name": "config",
        "args": {
          "config_path": "/var/run/secrets/.../",
          "build": {
            "config_file": "/etc/osbs/osbs-prod.conf",
            "platforms": [
              "x86_64",
              "ppc64le"
            ]
          }
        }
      },
      {
        "name": "inspect_parent",
      },
      {
        "name": "bump_release"
      },
      {
        "name": "add_labels_in_dockerfile",
        "args": {
          "labels": {
            "vendor": "...",
            "authoritative-source-url": "...",
            "distribution-scope": "...",
          }
        }
      },
      {
        "name": "add_filesystem",
        "args": {
          "koji_hub": "...",
          "repos": [...],
          "architectures": [
            "x86_64",
            "ppc64le"
          ]
        }
      }
    ],
    "buildstep_plugins": [
      {
        "name": "orchestrate_build"
      }
    ],
    "prepublish_plugins": [],
    "postbuild_plugins": [
      {
        "name": "fetch_worker_metadata"
      },
      {
        "name": "compare_components"
      },
      {
        "name": "tag_from_config",
        "args": {
          "tag_suffixes": {
            "unique": [
              "target-20170123055916"
            ],
            "primary": [
              "{version}-{release}",
              "{version}",
              "latest",
              "v3"
            ]
          }
        }
      },
      {
        "name": "group_manifests",
        "args": {
          "goarch": {
            "x86_64": "amd64"
          },
          "registries": ...
        }
      },
      {
        "name": "pulp_tag",
        "args": {
          "pulp_registry_name": ...,
          "pulp_secret_path": ...,
          ...
        }
      },
      {
        "name": "pulp_sync",
        "args": {
          "publish": false,
          "pulp_registry_name": ...,
          "pulp_secret_path": ...,
          ...
        }
      }
    ],
    "exit_plugins": [
      {
        "name": "pulp_publish",
        "args": {
          "pulp_registry_name": "...",
          "docker_registry": "..."
        }
      },
      {
        "name": "koji_import",
        "args": {
          "kojihub": ...,
          ...
        }
      },
      {
        "name": "delete_from_registry"
        "args": {
          "registries": { ... }
      },
      {
        "name": "store_metadata_in_osv3",
        "args": {"url": "...", ...}
      },
      {
        "name": koji_tag_build",
        "args": {
          "kojihub": ...,
          ...
        }
      }
    ]
  }

add_labels_in_dockerfile
~~~~~~~~~~~~~~~~~~~~~~~~

This existing plugin runs prior to add_filesystem in order to
correctly handle the case where a Dockerfile has no release label and
an orchestrator build has been created using a 'release' parameter to
set the value.

add_filesystem
~~~~~~~~~~~~~~

New parameter ``architectures``. This is used to fill in the
``arches`` parameter for ``image-build.conf``. When set, this new
parameter tells the plugin only to create (and wait for) the Koji
task, not to import its output files. That step is performed in the
worker builds.

inspect_parent
~~~~~~~~~~~~~~

This new plugin fetches the parent image's environment variables. The
environment variables are used by the ``bump_release`` plugin, which
may need them when processing the ``release`` label.

orchestrate_build
~~~~~~~~~~~~~~~~~

This existing buildstep plugin provides the core functionality of the
orchestrator build, but will need some changes for multi-platform
builds.

1. Look for a git repository file (``container.yaml``) and apply the
   ``platforms.only`` and ``platforms.not`` keys in it to its
   platforms parameter
2. Iterate over remaining platforms, and choose a worker cluster for
   each platform (see :ref:`config.yaml-clusters` for more details of
   how this is performed)
3. Create a build on each selected cluster by using the
   ``create_worker_build`` osbs-client API method, providing
   "platform", "release", and "koji_upload_dir" parameters
4. Monitor each created build, relaying streamed logs from
   get_build_logs(decode=True). If any worker build fails, the
   orchestrator build should also fail (once all builds complete).
5. Once all worker builds complete, for those that succeed fetch their
   annotations to discover their image manifest digests

Regarding relaying worker build logs see :ref:`logging`.

The return value of the plugin will be a dictionary of platform name
to BuildResult object.

fetch_worker_metadata
~~~~~~~~~~~~~~~~~~~~~

The new post-build plugin fetches metadata fragments from each worker
build using `get_config_map`_ (see :ref:`Metadata Fragment Storage`) and
makes it available to the `compare_components`_ and `koji_import`_
plugins.

It makes the metadata available by returning it from its run method in
the form of a dict, with each key being a platform name and each value
being the metadata fragment as a dict object.

This plugin will not be run for scratch builds.

compare_components
~~~~~~~~~~~~~~~~~~

This new post-build plugin analyses metadata fragments from each
worker build (see :ref:`Metadata Fragment Storage`) to find out the
RPM components installed in each image (name-version-release, and RPM
signatures), and will fail if there are any mismatches.

This plugin will not be run for scratch builds.

tag_from_config
~~~~~~~~~~~~~~~

This existing post-build plugin will run in the orchestrator build.

In addition it will take a new "tag_suffixes" parameter, which
osbs-client will populate with the unique and primary tag suffixes to
use.

It will perform parameter substitution on the primary tag suffixes,
using labels defined in the Dockerfile (in combination with the
environment variables from the parent image).

For the orchestrator build it will be given:
- a unique tag suffix (without a platform name)
- the strings ``{version}-{release}``, ``{version}`` and ``latest``
- any tag suffixes defined in additional-tags in the git repository

For the worker build it will be given:
- a unique tag suffix (with a platform name)

group_manifests
~~~~~~~~~~~~~~~

This new post-build plugin creates the `Docker Manifest List`_ in the
registry. It does this by inspecting the return value from the
orchestrate_build plugin to find the image manifest digests from the
platform-specific images, and assembling those together with platform
specifiers ('os' being 'linux', and 'architecture' being the GOARCH
for the platform) for each.

.. _`Docker Manifest List`: https://docs.docker.com/registry/spec/manifest-v2-2/#manifest-list

It takes the same "registries" parameter as the ``tag_and_push`` plugin,
as well as some others:

registries
  a mapping, each item having a key which is the push URL and a value
  which is a mapping, 'version' indicating the registry API version
  and optionally 'secret' indicating the name of the Kubernetes secret
  holding authentication credentials, for example::

    {
      "registry.example.com/repository": {
        "version": "v2",
        "secret": "registry-secret"
      }
    }

  Registries with version "v1" are ignored.

goarch
  a mapping, each item having a platform name as the key and the
  equivalent GOARCH architecture name as the value; this is built from
  the "platform:..." sections in the osbs.conf

group
  This optional Boolean parameter can be set to false in order to
  change the way this plugin works: instead of creating manifest
  lists, only tag the image manifest created by the worker build whose
  platform has GOARCH amd64. This option is only expected to be used
  during development.

It takes an optional parameter ``goarch`` which is a dict of
architecture names in the format used by the golang GOARCH variable,
indexed by platform name. It is optional (and if provided may not
provide GOARCH values for all platforms) because in the absence of a
GOARCH value for a given platform name, it will be assumed that the
platform name is already a valid GOARCH value.

Typically, a platform name might be "x86_64", whereas the
corresponding GOARCH value would be "amd64".

It needs to construct platform identifiers for each image manifest
digest. It uses these values:

os
  "linux"

architecture
  look up the platform name in the ``goarch`` dict passed as a
  parameter for the plugin and use the resulting value if found,
  otherwise use the platform name

The plugin's return value will include the manifest digest for the
created object.

pulp_tag
~~~~~~~~

This new post-build plugin, only run when API v1 support is enabled,
will set tags on the API v1 image that were uploaded by a worker
build.

pulp_sync
~~~~~~~~~

This existing post-build plugin will operate as before but with the
addition of an optional keyword parameter "publish" which defaults to
true.

The "publish" parameter will be set to false to tell the plugin not to
publish content, as that functionality is now in the pulp_publish
plugin.

pulp_publish
~~~~~~~~~~~~

This new exit plugin is for publishing content in the Pulp repository.

However, if any worker build failed, or the build was cancelled, this
plugin should instead remove the "v1" images from the Pulp repository.

koji_import
~~~~~~~~~~~

This new exit plugin replaces koji_promote. No longer responsible for
uploading the image tar archives (see `koji_upload`_), this plugin
creates a Koji build when the images all built successfully.

To do this it gathers the platform-specific metadata fragments created
by each worker build (see `koji_upload`_) and combines them. In
combining them, it takes care to make each buildroot ID unique but
preserving references to buildroots in the outputs.

The combined metadata fragments are then augmented with metadata
relating to the multi-platform build as a whole.

Logs for the builds are collected by inspecting the return value of
the `orchestrate_build`_ plugin. These logs are uploaded to Koji and
included in the build metadata as log outputs.

Finally the Koji API will be used to import the Koji Build.

This plugin will not be run for scratch builds.

delete_from_registry
~~~~~~~~~~~~~~~~~~~~

This existing exit plugin is no longer run in the worker
build. Instead it runs in the orchestrator build and deletes:

- the images pushed to the registry by worker builds, using the image
  manifest digests from their annotations

- the manifest list created in the registry by the `group_manifests`_
  plugin

koji_tag_build
~~~~~~~~~~~~~~

As previously, this plugin tags the Koji build created by the
"koji_promote" or "koji_import" plugins.

This plugin will not be run for scratch builds.

remove_worker_metadata
~~~~~~~~~~~~~~~~~~~~~~

This new exit plugin removes metadata fragments created by the worker
builds (see :ref:`Metadata Fragment Storage`).

This plugin will not be run for scratch builds.

Annotations/labels on orchestrator build
----------------------------------------

The orchestrator build will fetch annotations from completed worker
builds and add them to its own annotations to aid metrics
reporting. The annotations will look as follows::

  metadata:
    labels:
      koji-build-id: ...
    annotations:
      repositories:
        primary:
        - registry.example.com/repository:1.0-2
        - registry.example.com/repository:1.0
        - registry.example.com/repository:latest
        - registry.example.com/repository:v3
        unique:
        - registry.example.com/repository:target-20170123055916
      worker-builds:
        x86_64:
          build:
            cluster-url: openshift_url of worker cluster
            namespace: default
            build-name: repo-branch-abcde-1
          digests:
          - registry: ...
            repository: ...
            tag: ...
            digest: ...
          ...
          plugins-metadata:
            timestamps:
              koji: ...
              ...
            durations:
              koji: ...
              ...
            errors: {}
        ppc64le:
          build:
            cluster-url: openshift_url of worker cluster
            namespace: default
            build-name: repo-branch-abcde-1
          digests:
          - registry: ...
            repository: ...
            tag: ...
            digest: ...
          ...
          repositories:
            primary:
            - ...
            unique:
            - ...
          plugins-metadata:
            timestamps:
              koji: ...
              ...
            durations:
              koji: ...
              ...
            errors: {}
      plugins-metadata: '{"timestamps": {"orchestrate_build": "...", ...},
        "durations": {"orchestrate_build": ..., ...}, "errors": {}}'

The existing koji-build-id label is a string representing the
resulting Koji Build ID. It is only present when Koji integration is
enabled.

The existing "repositories" annotation holds a map with keys:

primary
  list of image pull specifications (for the manifest list) using
  primary tags

unique
  list of image pull specifications (for the manifest list) using
  unique tags

There is a new annotation:

worker-builds
  map of information about each worker build by platform

For each value in the worker-builds map:

build
  the server URL, namespace, and build name used for this worker build

digests
  the output in the registry (or Pulp, if Pulp integration is
  enabled), taken from the worker build's own digests build annotation

plugins-metadata
  the performance data of the worker build, taken from the worker
  build's own plugins-metadata build annotation

Note that annotations are in fact strings. The objects shown above are
really JSON-encoded when stored as annotations.

Anatomy of a worker build
-------------------------

Below is an example of the ATOMIC_REACTOR_PLUGINS environment variable
for a worker build::

  {
    "prebuild_plugins": [
      {
        "name": "add_filesystem",
        "args": {
          "koji_hub": "...",
          "from_task_id": "{koji_task_id}"
        }
      },
      {
        "name": "pull_base_image",
        "args": {
          "parent_registry": "..."
        }
      },
      {
        "name": "add_labels_in_dockerfile",
        "args": {
          "labels": {
            "vendor": "...",
            "authoritative-source-url": "...",
            "distribution-scope": "...",
            "release": "..."
          }
        }
      },
      {
        "name": "change_from_in_dockerfile"
      },
      {
        "name": "add_help"
      },
      {
        "name": "add_dockerfile"
      },
      {
        "name": "distgit_fetch_artefacts",
        "args": {
          "command": "rhpkg sources"
        }
      },
      {
        "name": "koji",
        "args": {
          "hub": "...",
          ...
        }
      },
      {
        "name": "add_yum_repo_by_url",
        "args": {
          "repourls": [...]
        }
      },
      {
        "name": "inject_yum_repo"
      },
      {
        "name": "distribution_scope"
      }
    ],
    "buildstep_plugins": [
      {
        "name": "docker_api"
      }
    ],
    "prepublish_plugins": [
      {
        "name": "squash"
      }
    ],
    "postbuild_plugins": [
      {
        "name": "all_rpm_packages"
      },
      {
        "name": "tag_from_config",
        "args": {
          "tag_suffixes": {
            "unique": ["target-20170123055916-x86_64"]
          }
        }
      },
      {
        "name": "tag_and_push",
        "args": {
          "registries": ...
        }
      },
      {
        "name": "pulp_push",
        "args": {
          "publish": false,
          "pulp_registry_name": ...
          ...
        }
      },
      {
        "name": "compress",
        "method": "gzip"
      },
      {
        "name": "koji_upload",
        "args": {
          "kojihub": "...",
          "upload_pathname": "...",
          "koji_upload_dir": "koji-upload/abc123",
          ...
        }
      }
    ],
    "exit_plugins": [
      {
        "name": "store_metadata_in_osv3"
        "args": {
          "url": "{url}"
        }
      },
      {
        "name": "remove_built_image"
      }
    ]
  }

This configuration is created by osbs-client's ``create_worker_build``
method, which has an optional ``filesystem_koji_task_id`` parameter
used for building base images.

tag_from_config (worker)
~~~~~~~~~~~~~~~~~~~~~~~~

See `tag_from_config`_ for full details of the way this existing
post-build plugin will be modified.

pulp_push
~~~~~~~~~

When Pulp integration and support for Docker Registry HTTP API V1 are
both enabled, this existing post-build plugin uploads the Docker image
archive so that Pulp is able to serve images using the V1 API (via
Crane).

A new optional Boolean parameter ``publish``, defaulting to true, tells
it whether to perform the "publish" operation (i.e. the dockpulp
``crane`` method) -- it should be set to false. This allows backwards
compatibility with how pulp_push was previously used.

koji_upload
~~~~~~~~~~~

This new post-build plugin uploads the image tar archive to Koji but
does not create a Koji build.

Additionally, it creates the platform-specific parts of the Koji build
metadata (see `Koji build`_) and places them in temporary storage
using `create_config_map`_ (see :ref:`Metadata Fragment
Storage`). Finally, it sets an annotation on its OpenShift Build
object indicating the name of the ConfigMap object.

The name it should choose for the ConfigMap object is its own
OpenShift Build name with the string "-md" concatenated onto it.

The metadata fragment will take the form of a JSON file::

  {
    "metadata_version": 0,
    "buildroots": [
      {
        "id": 1,
        "host": {
          "os": "Fedora 25",
          "arch": "x86_64"
        },
        "content_generator": {
          "name": "atomic-reactor",
          "version": "..."
        },
        "container": {
          "type": "docker",
          "arch": "x86_64"
        },
        "tools": [ ... ],
        "components": [
          {
            "name": "glibc",
            "version": "...",
            "release": "...",
            "epoch": "...",
            "arch": "...",
            "sigmd5": "...",
            "signature": "..."
          },
          {
            "name": "python",
            ...
          },,
          {
            "name": "atomic-reactor",
            ...
          },
          ...
        ]
      }
    ],
    "output": [
      {
        "buildroot_id": 1,
        "type": "docker-image",
        "arch": "x86_64",
        "filename": "docker-...-x86_64.tar.xz",
        "filesize": ...,
        "checksum_type": "md5",
        "checksum": ...,
        "extra": {
          "docker": {
            "id": "... (the image ID) ...",
            "parent_id": "... (the parent image's ID) ...",
            "repositories": [
              "some-registry/some-repository:unique-tag-x86_64",
              "some-registry/some-repository@sha256:(schema1digest)",
              "some-registry/some-repository@sha256:(schema2digest)"
            ],
            "tags": ["unique-tag-x86_64"]
          }
        },
        "components": [
          {
            "name": "glibc",
            "version": "...",
            "release": "...",
            "epoch": "...",
            "arch": "...",
            "sigmd5": "...",
            "signature": "..."
          },
          ...
        ]
      }
    ]
  }

metadata_version
  this is an integer corresponding to the metadata version this is a
  fragment of, i.e. 0

buildroots
  This is a list with a single item, a map. Of interest in that map:

  id
    This can be any value as long as it matches that used in the
    output map (see below). When koji_import combines metadata
    fragments together it will change the buildroot_id values in each
    fragment so that they outputs and buildroots match but have
    different values.

  components
    This is a list of RPMs available within the worker build's own
    container, and is assembled by querying the RPM database

output
  This is a list with a single item, a map. Of interest in that map:

  buildroot_id
    This must match the id used for the buildroots entry

  components
    This is a list of RPMs available within the built image, and is
    assembled by running an RPM database query within a container from
    that image (this is performed by the "all_rpm_packages" plugin,
    which runs before koji_upload)

This plugin will not be run for scratch builds.

store_metadata_in_osv3
~~~~~~~~~~~~~~~~~~~~~~

This existing exit plugin will store the `metadata_fragment`_
annotation using the result of the `koji_upload`_ plugin.

Annotations/labels on worker build
----------------------------------

The worker build annotations remain largely unchanged for
multi-platform builds. However, to support :ref:`Metadata Fragment
Storage`, new annotations will be added for non-scratch builds::

  metadata:
    labels:
      ...
    annotations:
      ...
      metadata_fragment: "configmap/build-name-7e4aab0-md"
      metadata_fragment_key: "metadata.json"
      help: "nil"

metadata_fragment
~~~~~~~~~~~~~~~~~

This annotation has a string value which is the kind and name of the
OpenShift object in which the metadata fragment is stored.

metadata_fragment_key
~~~~~~~~~~~~~~~~~~~~~

This is the key within the OpenShift object; as we are using ConfigMap
objects for this, it is the ConfigMap key whose value is the metadata
fragment (as a JSON string).

help
~~~~

This value, a JSON string, will indicate the result of the add_help
plugin. It is exactly analagous to the build.extra.image.help key for
the Koji Content Generator Metadata.

If the plugin did not run, the annotation will not be present.

If the plugin ran and found a source help file, the annotation will be
set to a JSON string representing the string value of the path to the
help file (e.g. ``"\"/path/to/file\""``).

If the plugin ran and found no source help file, the annotation will
be set to a JSON string representing a nil value (i.e. ``nil``).

Koji metadata
-------------

There are two Koji objects to consider: the task representing the
action of building the image, and the build representing the outputs.

Koji task
~~~~~~~~~

The "result" of a Koji task is a text field. For buildContainer tasks
this is used to store JSON data in and pyrpkg knows how to decode this
into a useful message including a URL to the resulting Koji build and
also a set of Docker pull specifications for the image.

The format remains unchanged:

koji_builds
  a list of Koji build IDs (although it will only have a single item)

repositories
  a list of fully-qualified pull specifications, with items relating
  to each tag (see :ref:`image-tags`)

The list of repositories will look no different than it did prior to
multi-platform support. However, each pull specification will relate
to a manifest list::

  {
    "koji_builds": [123456],
    "repositories": [
      "pulp-docker1/img/name:target-20170123055916",
      "pulp-docker1/img/name:1.0-2",
      "pulp-docker1/img/name:1.0",
      "pulp-docker1/img/name:latest"
    ]
  }

Note that only tags are included here as these are for convenience for
image owners. Manifest digests are included in the `Koji build`_, not
the Koji task.

.. _`Koji task logs`:

Koji task logs
''''''''''''''

The Koji build will have separate log files for each worker build, as
well as the orchestrator build's own log file. This is arranged
between the orchestrate_build plugin and the koji_promote/koji_import
plugin.

The logs from the orchestrator build will include output from the
orchestrate_build plugin indicating URLs for the worker builds from
which logs may be streamed.

It is up to the koji-containerbuild plugin to stream logs from those
URLs into separate output files for the Koji task.

In detail:

orchestrator.log
  Logs streamed from the orchestrator OpenShift build

*platform*.log
  Logs from the worker OpenShift build for the *platform*, obtained by
  demultiplexing the streamed orchestrator build logs

Koji build
~~~~~~~~~~

Koji builds will have entries in the output list as follows:

- One "docker-image" entry for each platform an image was built
  for, including:

  * an "arch" field

  * the docker pull-by-digest specifications for the distinct tag used
    by this platform-specific image manifest for both v2 schema 1 and
    v2 schema 2

  * the buildroot ID for the builder image used for this worker build

- One "log" entry for each *platform* an image was built for, including
  an "arch" field, with name *platform*.log -- the content of this
  file comes from having streamed the logs from the worker build,
  i.e. no additional log fetch is required

- One additional "log" entry for the logging output from the
  orchestrator build, with name orchestrator.log (*Note* this is a
  change from the existing name openshift-final.log) -- the content of
  this file comes from filtering the result of
  get_orchestrator_build_logs() from the orchestrator build.

The build metadata (build.extra.image) will have an additional key to
hold a pull-by-digest specification for the manifest list.

index
    information about the manifest list holding the images. This is a
    map with the following keys:

    pull
        list of pull specifications (as strings); one must include a
        tag, one must include a digest

    tags
        list of tags that were updated to point to this manifest list,
        one of which must be the one used in "pull"

Example::

  # This section is metadata for the build as a whole
  build:
    # usual name, version, release, source, time fields
    extra:
      image:
        # usual fields for OSBS builds: autorebuild, help
        # but also this new field describing the manifest list:
        index:
          pull:
          - pulp-docker01:8888/img:7.3-1
          - pulp-docker01:8888/img@sha256:1a2b3c4d5e...
          tags:
          - 7.3-1
          - 7.3
          - latest

  # This section is for metadata about atomic-reactor
  buildroots:
  - id: 1
    container:
      arch: x86_64
      type: docker
    # RPMs in x86_64 atomic-reactor container (from builder image)
    components:
    - name: glibc
      arch: x86_64
      ...

    - id: 2
    container:
      arch: ppc64le
      type: docker
    # RPMs in ppc64le atomic-reactor container (from builder image)
    components:
    - name: glibc
      arch: ppc64le
      ...

  # This section is for metadata about the built images
  output:
  - type: log
    # Top-level log output, as before; will not include output from worker builds, only orchestration.
    filename: orchestrate.log

  - type: log
    arch: x86_64
    filename: x86_64.log

  - type: log
    arch: ppc64le
    filename: ppc64le.log

  - type: docker-image
    arch: x86_64
    buildroot_id: 1
    filename: img-docker-7.3-1-x86_64.tar.gz
    extra:
      docker:
        id: sha256:abc123def...
        parent_id: sha256:123def456...
        repositories:
        - pulp-docker01:8888/img:20170601000000-2a892-x86_64
        - pulp-docker01:8888/img@sha256:789def567...
        - pulp-docker01:8888/img@sha256:123456789...
        # This pull specification refers to the image manifest for the x86_64 platform.
        tags:
        - 20170601000000-2a892-x86_64
        config:
          # docker registry config object
          docker_version: ...
          config:
            labels: ...
          ...

  - type: docker-image
    arch: ppc64le
    buildroot_id: 2
    filename: img-docker-7.3-1-ppc64le.tar.gz
    extra:
      docker:
        id: sha256:bcd234efg...
        parent_id: sha256:234efg567...
        repositories:
        - pulp-docker01:8888/img:20170601000000-ae58f-ppc64le
        - pulp-docker01:8888/img@sha256:890efg678...
        - pulp-docker01:8888/img@sha256:567890123...
        # This pull specification refers to the image manifest for the ppc64le platform.
        tags:
        - 20170601000000-ae58f-ppc64le
        config:
          # Docker registry config object
          docker_version: ...
          config:
            labels: ...
          ...
