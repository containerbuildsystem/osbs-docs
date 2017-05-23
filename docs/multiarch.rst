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

The cluster which will run orchestrator builds is referred to here as
the *orchestrator cluster*, and the cluster which will run worker
builds is referred to here as the *worker cluster*.

Note that the orchestrator cluster itself may also be configured to
accept worker builds, so the cluster may be both orchestrator and
worker. Alternatively some sites may want to have complete clusters
for each platform as well as for the orchestration.

Technically, only compute nodes in the required architectures are
needed to perform the container-building step. These can be arranged
either in a single mixed-architecture cluster, or with multiple
single-architecture clusters for each architecture, or as a mix of the
two.

The orchestrator build will make use of :ref:`config.yaml` to discover
which worker clusters to direct builds to and whether/which `node
selector`_ is required for each.

.. _`node selector`: https://docs.openshift.org/latest/admin_guide/managing_projects.html#developer-specified-node-selectors

Orchestration required for multi-platform builds
------------------------------------------------

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
    `Temporary Storage`_)

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

- Tag and push the image to the container registry (see `Tagging`_)

- Query the image to discover installed RPM packages (by running
  ``rpm`` inside it)

- If Koji integration is enabled:

  - Upload image tar archive to Koji but do not create a Koji Build

  - Upload the Koji metadata fragment (buildroot information, built
    image information) to temporary storage (see `Temporary Storage`_)

- Update this OpenShift Build with annotations about output,
  performance, errors, etc

.. graphviz:: images/multi-during-build.dot
   :caption: During build

.. graphviz:: images/multi-after-build.dot
   :caption: After build

Temporary Storage
-----------------

When creating a Koji Build, the `koji_promote`_ plugin needs to
assemble Koji Build Metadata, including:

- components installed in each builder image (worker builds and
  orchestrator build)

- components installed in each built image

- information about each build host

To assist the orchestrator build in assembling this (JSON) data, the
worker builds will gather information about their build hosts, builder
images, and built images. They then need to pass this data to the
orchestrator build. After creating the Koji Build, the orchestrator
build must then free any resources used in passing the data.

A number of approaches are possible for this, detailed below.

Create OpenShift object
~~~~~~~~~~~~~~~~~~~~~~~

The worker build could create an OpenShift object (perhaps a Secret or
ConfigMap) in the worker cluster and store the name of this object in
its OpenShift Build annotations. To do this the worker cluster's
"builder" service account will need to be granted permission to create
objects of the appropriate type.

The orchestrator build would then be responsible for removing the
OpenShift object from the worker cluster. To do this, the worker
cluster's "orchestrator" service account will need to be granted
permission to get and delete objects of the appropriate type.

Store a blob in the Docker registry
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The worker build could store its JSON fragment as a blob in the Docker
repository it pushed to. This blob would not be referenced by any
manifest. On build completion, the worker build would store the blob
digest for this JSON fragment in an OpenShift Build annotation for the
orchestrator build to inspect.

The orchestrator build would be able to discover the blob digests,
fetch them from the registry and put together the metadata.

Afterwards, even on error, it would delete them.

Between being created by the worker build and deleted by the
orchestrator build, this blob would be "dangling" i.e. not referenced
by any manifest. If the docker/distribution garbage collector is run
during this time the blob will be removed.

Upload file to Koji hub
~~~~~~~~~~~~~~~~~~~~~~~

In the same way the Docker image archive is uploaded to the Koji hub,
the JSON fragment could also be uploaded. However, it would not be
referenced in the Koji Build as an output.

**Will this be garbage collected?**

Submitting builds
-----------------

A new optional parameter ``--arches`` will be added to the
``container-image`` subcommand provided by pyrpkg. If supplied,
``--scratch`` must also be supplied. It will pass the parameter
``arches`` to the Koji task (implemented by the
``koji-containerbuild`` plugin for Koji).

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

Excluding platforms
-------------------

Some container images will need to be built for multiple platforms but
some may not.

The full set of platforms for which builds may be required will come
initially from the Koji build tag associated with the build target, or
from the ``platforms`` parameter provided to the
``create_orchestrator_build`` API method when Koji is not used.

A configuration file present in the git repository named
``container.yaml`` may contain configuration keys relevant to platform
selection.

This set of platforms can be reduced in various ways:

- Explicit subset:

  * container image builds can be submitting with a parameter
    ``--arches``, overriding the set of platforms specified by the Koji
    build target, in the same way as for building RPM packages

  * the ``container.yaml`` configuration file's ``platforms.only`` key
    can further restrict this set of platforms (via set union)

- Excluding platforms:

  * the ``container.yaml`` configuration file's ``platforms.not`` key
    can restrict the set of platforms even further, by removing
    specific platforms from those remaining

Tagging
-------

Image manifests (from worker builds)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The image manifests will be tagged using a unique tag include the
timestamp and platform name.

Manifest lists (from orchestrator build)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The manifest list will be tagged using:

- ``latest``
- ``$version`` (the ``version`` label)
- ``$version-$release`` (the ``version`` and ``release`` labels together)
- a unique tag including the timestamp
- any additional tags configured in the git repository

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

Git Configuration
-----------------

Each git repository to build from may contain a ``container.yaml``
file in the following format::

  platforms:
    # all these keys are optional

    only:
    - x86_64   # can be a list (as here) or a string (as below)
    - ppc64le
    - armhfp
    not: armhfp

platforms
~~~~~~~~~

Keys in this map relate to multi-platform builds.

only
  list of platform names (or a single platform name as a string); this
  will be combined with the ``platforms`` parameter to the
  `orchestrate_build`_ plugin using set union

not
  list of platform names (or a single platform name as a string);
  platforms named here will be removed from the ``platforms``
  parameter to the `orchestrate_build`_ plugin using set difference

Client Configuration
--------------------

The osbs-client configuration file format will be augmented with
instance-specific field ``node_selector``.

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

Example configuration file: Koji builder
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The configuration required for submitting an orchestrator build is
different than that required for the orchestrator build itself to
submit worker builds. The ``osbs.conf`` used by the Koji builder would
include::

  [general]
  build_json_dir = /usr/share/osbs/
  
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

  # This causes koji_promote not to be configured, and for the low
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

  # This node selector, combined with the platform-specific node
  # selector, will be applied to worker builds.
  low_priority_node_selector = lowpriority=true

  # and auth options, registries, secrets, etc
  
  [prod-osd]
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

Client API changes
------------------

Two new API methods will handle orchestration, and the existing API
method for creating builds will gain a new optional parameter.

create_orchestrator_build
~~~~~~~~~~~~~~~~~~~~~~~~~

This will take the same parameters as ``create_prod_build`` (except
for platform) but will use different templates to create the
BuildConfig (``orchestrator.json`` and
``orchestrator_inner.json``). The orchestrator BuildConfig template
will set its resource request.

Instead of a ``platform`` parameter specifying a single platform it
will take a ``platforms`` parameter, which is a list of platforms to
create worker builds for. The ``koji-containerbuild`` plugin for Koji
will supply this parameter from the list of architectures configured
for the Koji build tag for the Koji build target the build is for.

This method takes an ``arrangement_version`` parameter to select
which arrangement of plugins is to be used in the orchestrator and
worker builds.

This method can only be used for cluster definitions that specify they
can orchestrate (see :ref:_`Can Orchestrate`).

create_worker_build
~~~~~~~~~~~~~~~~~~~

This will have required parameters:

platform
  the platform to build for

release
  the value to use for the release label

as well as the optional parameters:

filesystem_koji_task_id
  Koji Task ID of image-build task

arrangement_version
  to select which arrangement of plugins is to be used in the orchestrator and
  worker builds

It will use different templates to create the BuildConfig
(``worker.json`` and ``worker_inner.json``). The worker BuildConfig
template will not set its resource request and will use the default
supplied by the worker cluster.

create_prod_build
~~~~~~~~~~~~~~~~~

This existing API method will gain an optional ``platform`` parameter
(the platform to build for) and will remain in place for compatibility
but can be removed once all site OSBS implementations are using
orchestration.

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
        "name": "add_filesystem",
        "args": {
          "koji_hub": "...",
          "repos": [...],
          "architectures": [
            "x86_64",
            "ppc64le"
          ]
        }
      },
      {
        "name": "inspect_parent",
      },
      {
        "name": "bump_release"
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
        "name": "compare_rpm_packages"
      },
      {
        "name": "group_manifests",
        "args": {
          "registries": ...
        }
      },
      {
        "name": "pulp_sync"
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
        "name": "pulp_pull"
      },
      {
        "name": "koji_promote",
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

reactor_config
~~~~~~~~~~~~~~

This plugin parses the atomic-reactor config and makes it available to
other plugins.

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

This plugin provides the core functionality of the orchestrator
build. It provides the following functionality:

1. Look for a git repository file (``container.yaml``) and apply the
   ``platforms.only`` and ``platforms.not`` keys in it to its
   platforms parameter
2. Iterate over remaining platforms, and choose a worker cluster for
   each platform (see :ref:`config.yaml-clusters` for more details of
   how this is performed)
3. Create a build on each selected cluster by using the
   ``create_worker_build`` osbs-client API method, providing
   "platform", and "release" parameters
4. Monitor each created build. If any worker build fails, the
   orchestrator build should also fail (once all builds complete).
5. Once all worker builds complete, fetch their logs and -- for those
   that succeeded -- their annotations to discover their image
   manifest digests

The return value of the plugin will be a dictionary of platform name
to BuildResult object.

compare_rpm_packages
~~~~~~~~~~~~~~~~~~~~

This new post-build plugin analyses log files from each worker build
to find out the RPM components installed in each image
(name-version-release, and RPM signatures), and will fail if there are
any mismatches. The ``all_rpm_packages`` plugin in the worker build
will be modified to log the RPM list in a parseable format to
facilitate this.

group_manifests
~~~~~~~~~~~~~~~

This new post-build plugin creates the Docker Manifest List in the
registry. It does this by inspecting the return value from the
orchestrate_build plugin to find the image manifest digests from the
platform-specific images.

The plugin's return value will include the manifest digest for the
created object.

pulp_publish
~~~~~~~~~~~~

This new exit plugin is for publishing content in the Pulp repository.

However, if any worker build failed, or the build was cancelled, this
plugin should instead remove the "v1" images from the Pulp repository.

koji_promote
~~~~~~~~~~~~

No longer responsible for uploading the image tar archives (see
`koji_upload`_), this exit plugin creates a Koji build when the images
all built successfully.

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

koji_tag
~~~~~~~~

As previously, this plugin tags the Koji build created by the
"koji_promote" plugin.

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
        - ...
        unique:
        - ...
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
  list of image pull specifications (across all worker builds) using
  primary tags

unique
  list of image pull specifications (across all worker builds) using
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
        "name": "dockerbuild"
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
        "name": "tag_by_labels"
      },
      {
        "name": "tag_from_config"
      },
      {
        "name": "tag_and_push",
        "args": {
          "registries": {
            "...": { "insecure": true }
          }
        }
      },
      {
        "name": "pulp_push",
        "args": {
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
          "upload_pathname": "..."
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

all_rpm_packages
~~~~~~~~~~~~~~~~

This existing post-build plugin will be modified. As well as fetching
the list of installed RPMs in the built image, it will emit this list
using a specially-formatted log output.

pulp_push
~~~~~~~~~

When Pulp integration and support for Docker Registry HTTP API V1 are
both enabled, this existing post-build plugin uploads the Docker image
archive so that Pulp is able to serve images using the V1 API (via
Crane).

koji_upload
~~~~~~~~~~~

This new post-build plugin uploads the image tar archive to Koji but
does not create a Koji build.

Additionally, it creates the platform-specific parts of the Koji build
metadata (see `Koji build`_) and places them in temporary storage (see
`Temporary Storage`_).

The metadata fragment will take the form of a JSON file::

  {
    "buildroots": [
      {
        ... single entry ...
      }
    ],
    "output": [
      {
        ... single entry ...
      }
    ]
  }

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
  to each tag (see `Tagging`_)

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

Koji task logs
''''''''''''''

The Koji build will have separate log files for each worker build, as
well as the orchestrator build's own log file. This is arranged
between the orchestrate_build plugin and the koji_promote plugin.

For the Koji task, however, it is also desirable to have separate logs
for each worker build (so that build submitters can watch each worker
build separately), but the orchestrator build is only able to stream a
single log file.

The solution is for the orchestrate_build plugin to emit
specially-formatted logs, tagged with the platform for which the
worker build is being performed, and for osbs-client to understand how
to separate these tagged log lines from the rest of the log output.

This is similar to the way the orchestrate_build plugin gathers image
component information from the worker builds (see
`all_rpm_packages`_).

Koji build
~~~~~~~~~~

Koji builds will have entries in the output list as follows:

- One "docker-image" entry for each platform an image was built
  for, including:

  * an "arch" field

  * the docker pull-by-digest specification for the distinct tag used
    by this platform-specific image manifest

  * the buildroot ID for the builder image used for this worker build

- One "log" entry for each platform an image was built for, including
  an "arch" field

- One additional "log" entry for the logging output from the
  orchestrator build

The build metadata (build.extra.image) will have an additional key to
hold a pull-by-digest specification for the manifest list.

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
        - pulp-docker01:8888/img@sha256:890efg678…
        # This pull specification refers to the image manifest for the ppc64le platform.
        tags:
        - 20170601000000-ae58f-ppc64le
        config:
          # Docker registry config object
          docker_version: ...
          config:
            labels: ...
          ...
