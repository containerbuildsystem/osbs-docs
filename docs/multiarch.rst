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

Technically, only compute nodes in the required architectures are
needed to perform the container-building step. These can be arranged
either in a single mixed-architecture cluster, or with multiple
single-architecture clusters for each architecture, or as a mix of the
two.

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
    `Metadata Fragment Storage`_)

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
    image information) to temporary storage (see `Metadata Fragment
    Storage`_)

- Update this OpenShift Build with annotations about output,
  performance, errors, etc

.. graphviz:: images/multi-during-build.dot
   :caption: During build

.. graphviz:: images/multi-after-build.dot
   :caption: After build

Metadata Fragment Storage
-------------------------

When creating a Koji Build, the `koji_import`_ plugin needs to
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

This data will be stored in OpenShift ConfigMap object in the worker
cluster using `create_config_map`_, and its name will be stored in the
OpenShift Build annotations for the worker build. To do this the
worker cluster's "builder" service account will need to be granted
permission to create ConfigMap objects.

The orchestrator build will collect the metadata fragment using
`get_config_map`_ when assembling the fragments together with the
platform-neutral metadata in `koji_import`_.

The orchestrator build is then responsible for removing the OpenShift
ConfigMap from the worker cluster using `delete_config_map`_. To do
this, the worker cluster's "orchestrator" service account will need to
be granted permission to get and delete ConfigMap objects.

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

.. _`Logging`:

Logging
-------

Logs from worker builds will be made available via the orchestrator
build, and clients (including koji-containerbuild) will be able to
separate individual worker build logs out from that log stream using
a new osbs-client API method.

Multiplexing
~~~~~~~~~~~~

In order to allow the client to de-multiplex logs containing a mixture
of logs from an orchestrator build and from its worker builds, a new
logging field, platform, is used. Within atomic-reactor all logging
should be done through a LoggerAdapter which adds this ``platform``
keyword to the ``extra`` dict passed into logging calls. These objects
should come from a factory function::

  def get_logger(name, platform=None):
      return logging.LoggerAdapter(logging.getLogger(name),
                                   {'platform': platform or '-'})

The logging format will include this new field::

  %(asctime)s platform:%(platform)s - %(name)s - %(levelname)s - %(message)s

resulting in log output like this::

  2017-06-23 17:18:41,791 platform:- - atomic_reactor.foo - DEBUG - this is from the orchestrator build
  2017-06-23 17:18:41,791 platform:x86_64 - atomic_reactor.foo - INFO - 2017-06-23 17:18:41,400 platform:- atomic_reactor.foo -  DEBUG - this is from a worker build
  2017-06-23 17:18:41,791 platform:x86_64 - atomic_reactor.foo - INFO - continuation line

Demultiplexing will be possible using a new osbs-client API method,
`get_orchestrator_build_logs`_. This method is a generator function
that returns objects with these attributes:

platform
  str, platform name if worker build, else None

line
  str, log line (Unicode)

See the example below for what this would look like for these sample
log lines.

In order to do this it should find the third space-separated field
from the log line. Since the asctime value contains a space between
the date and time, the third field is for the platform.

If the platform field does not start ``platform:``, then this is a log
line from the orchestrator build which has (mistakenly) not been
logged using the adapter.

If the platform field matches ``platform:-``, then this is a log line
from the orchestrator build.

Otherwise, the platform name can be found by removing the
``platform:`` prefix.

For orchestrator build logs, the line is returned as-is.

For worker build logs, the wrapping orchestrator log fields
('timestamp', 'platform', 'name', and 'levelname' fields) are dropped
leaving only the worker log line (the 'message' field).

This message field is then parsed as log fields. If the third field of
the worker build log line matches ``platform:-`` it is removed;
otherwise the line is left alone.

See the example below to see this illustrated.

Encoding issues
~~~~~~~~~~~~~~~

When retrieving logs from containers, the text encoding used is only
known to the container. It may be based on environment variables
within that container; it may be hard-coded; it may be influenced by
some other factor. For this reason, container logs are treated as byte
streams.

This applies to:

- containers used to construct the built image
- the builder image running atomic-reactor for a worker build
- the builder image running atomic-reactor for an orchestrator build

When retrieving logs from a build, OpenShift cannot say which encoding
was used. However, atomic-reactor can define its own output encoding
to be UTF-8. By doing this, all its log output will be in a known
encoding, allowing osbs-client to decode it. To do this it should call
``locale.setlocale(locale.LC_ALL, "")`` and the Dockerfile used to
create the builder image must set an appropriate environment
variable::

  ENV LC_ALL=en_US.UTF-8

In this way, the osbs-client ``get_build_logs`` method will once again
be able to return an iterable of decoded strings, rather than of a
bytes type. It should gain a new keyword parameter ``decode`` with
default value False. When ``decode=True`` is supplied, older
osbs-client versions will fail with TypeError and the caller must
inspect the type of the returned objects.

Orchestrator builds want to retrieve logs from worker builds, then
relay them via logging. By knowing that the builder image for the
worker is the same as the builder image for the orchestrator, we also
know the encoding for those logs to be UTF-8.

One final issue is that the build logs from the Docker Python API must
be in a known encoding. Previously this API returned a byte stream
containing JSON objects describing the logs. However, by supplying
``decode=True`` to the Docker Python API's ``build`` method, we can get
a generator of decoded dicts as its return value. (The Docker Python
API assumes UTF-8, but uses a 'replace' errors handler.)

Example
~~~~~~~

Here is an example Python session demonstrating this interface::

  >>> server = OSBS(...)
  >>> logs = server.get_orchestrator_build_logs(...)
  >>> [(item.platform, item.line) for item in logs]
  [(None, '2017-06-23 17:18:41,791 platform:- - atomic_reactor.foo - DEBUG - this is from the orchestrator build'),
   ('x86_64', '2017-06-23 17:18:41,400 atomic_reactor.foo - DEBUG - this is from a worker build'),
   ('x86_64', 'continuation line')]

Note:

- the lines are string objects (Unicode), not bytes objects

- the orchestrator build's logging fields have been removed from the
  worker build log line

- the "outer" orchestrator log fields have been removed from the
  worker build log line, and the ``platform:-`` field has also been
  removed from the worker build's log line

- where the worker build log line had no timestamp (perhaps the log
  line had an embedded newline, or was logged outside the adapter
  using a different format), the line was left alone

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

See `Logging`_ for more details.

create_worker_build
~~~~~~~~~~~~~~~~~~~

This existing API method will gain an additional optional parameter:

filesystem_koji_task_id
  Koji Task ID of image-build task

This will be supplied as a "from_task_id" argument to the
add_filesystem plugin in the worker build.

create_config_map
~~~~~~~~~~~~~~~~~

This new API method will be used by a worker build to create a
ConfigMap object in which the metadata fragment for the image build
will be stored. It takes two parameters:

name
  This string is the name of the ConfigMap object to create

data
  This is a dict whose keys and values should be stored in the
  ConfigMap. For `Metadata Fragment Storage`_ it is expected that the
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
   "platform", and "release" parameters
4. Monitor each created build, relaying streamed logs from
   get_build_logs(decode=True). If any worker build fails, the
   orchestrator build should also fail (once all builds complete).
5. Once all worker builds complete, for those that succeed fetch their
   annotations to discover their image manifest digests

Regarding relaying worker build logs see :ref:`Logging`.

The return value of the plugin will be a dictionary of platform name
to BuildResult object.

fetch_worker_metadata
~~~~~~~~~~~~~~~~~~~~~

The new post-build plugin fetches metadata fragments from each worker
build using `get_config_map`_ (see `Metadata Fragment Storage`_) and
makes it available to the `compare_rpm_packages`_ and `koji_import`_
plugins.

It makes the metadata available by returning it from its run method in
the form of a dict, with each key being a platform name and each value
being the metadata fragment as a dict object.

compare_rpm_packages
~~~~~~~~~~~~~~~~~~~~

This new post-build plugin analyses metadata fragments from each
worker build (see `Metadata Fragment Storage`_) to find out the RPM
components installed in each image (name-version-release, and RPM
signatures), and will fail if there are any mismatches.

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

koji_import
~~~~~~~~~~~~

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

koji_tag
~~~~~~~~

As previously, this plugin tags the Koji build created by the
"koji_promote" or "koji_import" plugins.

remove_worker_metadata
~~~~~~~~~~~~~~~~~~~~~~

This new exit plugin removes metadata fragments created by the worker
builds (see `Metadata Fragment Storage`_).

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
metadata (see `Koji build`_) and places them in temporary storage
using `create_config_map`_ (see `Metadata Fragment
Storage`_). Finally, it sets an annotation on its OpenShift Build
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
              "some-registry/some-repository:tag",
              "some-registry/some-repository@sha256:(digest)"
            ]
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

store_metadata_in_osv3
~~~~~~~~~~~~~~~~~~~~~~

This existing exit plugin will store the `metadata_fragment`_
annotation using the result of the `koji_upload`_ plugin.

Annotations/labels on worker build
----------------------------------

The worker build annotations remain largely unchanged for
multi-platform builds. However, to support `Metadata Fragment
Storage`_ a new annotation will be added::

  metadata:
    labels:
      ...
    annotations:
      ...
      metadata_fragment: "configmap/build-name-7e4aab0-md"
      metadata_fragment_key: "metadata.json"

metadata_fragment
~~~~~~~~~~~~~~~~~

This annotation has a string value which is the kind and name of the
OpenShift object in which the metadata fragment is stored.

metadata_fragment_key
~~~~~~~~~~~~~~~~~~~~~

This is the key within the OpenShift object; as we are using ConfigMap
objects for this, it is the ConfigMap key whose value is the metadata
fragment (as a JSON string).

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

  * the docker pull-by-digest specification for the distinct tag used
    by this platform-specific image manifest

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
        tag, one may include a digest

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
