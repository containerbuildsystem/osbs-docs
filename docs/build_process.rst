.. _`build process`:

Understanding the Build Process
===============================

OSBS creates an OpenShift BuildConfig to store the configuration
details about how to run atomic-reactor and which git commit to build
from. If there is already a BuildConfig for the image (and branch), it
is updated. Afterwards, a Build is instantiated from the BuildConfig.

The default OpenShift BuildConfig runPolicy of "Serial" is used,
meaning only one Build will run at a time for a given BuildConfig, and
other Builds will remain queued and unprocessed until the running
build finishes. This is preferable to "SerialLatestOnly", which
cancels all but the most recent pending queued build, because build
logs for any user-submitted build can be watched through to
completion.

OSBS uses two types of OpenShift Build:

*worker build*
    for creating the container images

*orchestrator build*
    for creating and managing worker builds

The cluster which runs orchestrator builds is referred to here as the
*orchestrator cluster*, and the cluster which runs worker builds is
referred to here as the *worker cluster*.

Note that the orchestrator cluster itself may also be configured to
accept worker builds, so the cluster may be both orchestrator and
worker. Alternatively some sites may want separate clusters for these
functions.

The orchestrator build makes use of :ref:`config.yaml` to discover
which worker clusters to direct builds to and whether/which `node
selector`_ is required for each.

.. _`node selector`: https://docs.openshift.org/latest/admin_guide/managing_projects.html#developer-specified-node-selectors

The separation of tasks between the orchestrator build and the worker
build is called the "arrangement".

*Note:*
    Arrangement versions lower than 6 are no longer supported
    (but each new version typically builds on the previous one,
    so no functionality is lost).

Arrangement version 1 (delegation)
----------------------------------

*This arrangement version is no longer supported.*

In this arrangement, the orchestrator build chooses a worker cluster
and creates a worker build in that cluster. From then on, the worker
build performs all the tasks required.

Delegating this way allows for future features such as multi-platform
builds (with multiple worker builds per orchestrator build) and
automated triggered rebuilds (where the orchestrator cluster has
complete knowledge of the latest build configurations and their
triggers).

.. graphviz:: images/arrangement-v1.dot
   :caption: Orchestrator and worker builds (arrangement v1)

Orchestrator
~~~~~~~~~~~~

Steps performed by the orchestrator build are:

- For layered images:

  * pull the parent image in order to inspect environment variables
    (this is to allow substitution to be performed in the "release"
    label)

- Supply a value for the "release" label if it is missing and Koji
  integration is enabled (this value is provided to the worker build)

- Apply labels supplied via build request parameter to the Dockerfile

- Parse server-side configuration for atomic-reactor in order to know
  which worker clusters may be considered

- Create a worker build on one of the configured clusters, and collect
  logs and status from it

- Update this OpenShift Build with annotations about output,
  performance, errors, worker builds used, etc

- Perform any clean-up required:

  * For layered images, remove the parent image which was pulled at
    the start

Worker
~~~~~~

The orchestration step will create an OpenShift build for performing
the worker build. Inside this, atomic-reactor will execute these steps
as plugins:

- Get (or create) parent layer against which the Dockerfile will run

  * For base images, the base image filesystem is created using Koji
    and imported as the initial image layer

  * For layered images, the FROM image is pulled from the source
    registry

- Supply a value for the "release" label if it is missing and Koji
  integration is enabled XXX seems wrong?

- Make various alterations to the Dockerfile

- Fetch any supplemental files required by the Dockerfile

- Build the container

- Squash the image so that only a single layer is added since the
  parent image

- Query the image to discover installed RPM packages (by running
  ``rpm`` inside it)

- Tag and push the image to the container registry

- Sync the image into Pulp from the container registry (if Pulp
  integration is enabled) and publish it -- in this case the image is
  deleted from the container registry later

- Compress the docker image tar archive

- Pull the image back from the Pulp registry in order to accurately
  record its image ID (required because Pulp lacks support for v2
  schema 2 manifests)

- If Koji integration is enabled, upload image tar archive to Koji and
  create a Koji Build

- Update this OpenShift Build with annotations about output,
  performance, errors, etc

- If Koji integration is enabled, tag the Koji build

- Send email notifications if required

- Perform any clean-up required:

  * Remove the parent image which was fetched or created at the start

  * If Pulp integration is enabled, delete the image from the
    container registry

Arrangement version 2 (orchestrator creates filesystem)
-------------------------------------------------------

*This arrangement version is no longer supported.*

This arrangement, available since osbs-client-0.41, is identical to
version 1 except that:

- There is a new plugin for verifying the parent image comes from a
  build that exists in Koji

- The add_filesystem plugin runs in the orchestrator build as well as
  the worker build. This is to create a single Koji "`image-build`_"
  task to create filesystems for all required architectures.

.. _`image-build`: https://docs.pagure.org/koji/image_build/

In the worker build, the add_filesystem still runs but does not create
a Koji task. Instead the orchestrator build tells it which Koji task
ID to stream the filesystem tar archive from. Each worker build only
streams the filesystem tar archive for the architecture it is running
on.

Arrangement version 3 (Koji build created by orchestrator)
----------------------------------------------------------

*This arrangement version is no longer supported.*

This arrangement builds on version 2. The ``koji_promote`` plugin,
which was previously responsible for creating the Koji build, is
replaced by these plugins:

- worker build

  * koji_upload

- orchestrator build

  * fetch_worker_metadata (see :ref:`Metadata Fragment Storage`)

  * koji_import

  * koji_tag_build

Additionally the ``sendmail`` plugin now runs in the orchestrator
build and not the worker build.

.. _`Metadata Fragment Storage`:

Metadata Fragment Storage
-------------------------

When creating a Koji Build using arrangement 3 and newer, the
koji_import plugin needs to assemble Koji Build Metadata, including:

- components installed in each builder image (worker builds and
  orchestrator build)

- components installed in each built image

- information about each build host

To assist the orchestrator build in assembling this (JSON) data, the
worker builds gather information about their build hosts, builder
images, and built images. They then need to pass this data to the
orchestrator build. After creating the Koji Build, the orchestrator
build must then free any resources used in passing the data.

The method used for passing the data from the worker builds to the
orchestrator build is to store it temporarily in a ConfigMap object in
the worker cluster. Its name is stored in the OpenShift Build
annotations for the worker build. To do this the worker cluster's
"builder" service account needs permission to create ConfigMap
objects.

The orchestrator build collects the metadata fragments and assembles
them together with the platform-neutral metadata in the koji_import
plugin.

The orchestrator build is then responsible for removing the OpenShift
ConfigMap from the worker cluster. To do this, the worker cluster's
"orchestrator" service account needs permission to get and delete
ConfigMap objects.

Arrangement version 4 (Multiple worker builds)
----------------------------------------------

*This arrangement version is no longer supported.*

This arrangement moves most of the Pulp integration work to the
orchestrator build, allowing for multiple worker builds. Only the
pulp_push plugin remains in the worker build.

.. _group-manifests:

A new plugin, group_manifests, creates a `manifest list`_ object in
the registry, grouping together the image manifests from the worker
builds.

.. _`manifest list`: https://docs.docker.com/registry/spec/manifest-v2-2/#manifest-list

.. graphviz:: images/arrangement-v4.dot
   :caption: Orchestrator and worker builds (arrangement v4)

Arrangement version 5 (ODCS Integration)
----------------------------------------

*This arrangement version is no longer supported.*

This arrangement builds on version 4. The ``resolve_composes`` plugin enables
integration with `odcs`_. See :ref:`yum-repositories-odcs-compose` for details.

.. _`odcs`: https://pagure.io/odcs

- worker build

  * No changes

- orchestrator build

  * resolve_composes

.. _`logging`:

Arrangement Version 6 (reactor_config_map)
------------------------------------------

In this arrangement version, environment parameters are provided by **reactor_config**.
The order of plugins is the same, but hard coded, or placeholder, environment
parameters in **orchestrator_inner** and **worker_inner** json files change.

An osbs-client configuration option **reactor_config_map** is required to define
the name of the ``ConfigMap`` object holding **reactor_config**. This
configuration option is mandatory for arrangement versions greater than or
equal to 6. The existing osbs-client configuration **reactor_config_secret**
is deprecated (for all arrangements).

For more details on how the build system is configured as of
Arrangement 6, consult the :ref:`build_parameters` document.

As of October 2019, Pulp is no longer supported in Arrangement 6 for either worker
or orchestrator builds.


Logging
-------

Logs from worker builds is made available via the orchestrator build,
and clients (including koji-containerbuild) are able to separate
individual worker build logs out from that log stream using an
osbs-client API method.

Multiplexing
~~~~~~~~~~~~

In order to allow the client to de-multiplex logs containing a mixture
of logs from an orchestrator build and from its worker builds, a
special logging field, platform, is used. Within atomic-reactor all
logging goes through a LoggerAdapter which adds this ``platform``
keyword to the ``extra`` dict passed into logging calls, resulting in
log output like this::

  2017-06-23 17:18:41,791 platform:- - atomic_reactor.foo - DEBUG - this is from the orchestrator build
  2017-06-23 17:18:41,791 platform:x86_64 - atomic_reactor.foo - INFO - 2017-06-23 17:18:41,400 platform:- atomic_reactor.foo - DEBUG - this is from a worker build
  2017-06-23 17:18:41,791 platform:x86_64 - atomic_reactor.foo - INFO - continuation line

Demultiplexing is possible using a the osbs-client API method,
``get_orchestrator_build_logs``, a generator function that returns
objects with these attributes:

platform
  str, platform name if worker build, else None

line
  str, log line (Unicode)

The "Example" section below demonstrates how to use the
``get_orchestrator_build_logs()`` method in Python to parse the above log
lines.

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

Orchestrator builds want to retrieve logs from worker builds, then
relay them via logging. By knowing that the builder image for the
worker is the same as the builder image for the orchestrator, we also
know the encoding for those logs to be UTF-8.

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

- the lines are (Unicode) string objects, not bytes objects

- the orchestrator build's logging fields have been removed from the
  worker build log line

- the "outer" orchestrator log fields have been removed from the
  worker build log line, and the ``platform:-`` field has also been
  removed from the worker build's log line

- where the worker build log line had no timestamp (perhaps the log
  line had an embedded newline, or was logged outside the adapter
  using a different format), the line was left alone


Autorebuilds
------------

OSBS’s autorebuild feature automatically starts new builds of layered images
whenever the base parent image changes. This is particularly useful for image
owners that maintain a large hierarchy of images, which would otherwise require
manually starting each image build in the correct order. Instead, image owners
can start a build for the topmost ancestor which upon completion triggers the
next level of layered images, and so on.

Builds may opt in to autorebuilds with an
:ref:`autorebuild entry in the dist-git configuration. <osbs-config-autorebuild>`
Additional options for autorebuilds can be configured in
:ref:`container.yaml. <container.yaml-autorebuild>`

