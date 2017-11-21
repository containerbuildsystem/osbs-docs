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

Arrangement version 1 (delegation)
----------------------------------

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

- Upload the image archive to Pulp (if Pulp integration and v1 support
  are both enabled)

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

This arrangement moves most of the Pulp integration work to the
orchestrator build, allowing for multiple worker builds. Only the
pulp_push plugin remains in the worker build.

A new plugin, group_manifests, creates a `manifest list`_ object in
the registry, grouping together the image manifests from the worker
builds.

.. _`manifest list`: https://docs.docker.com/registry/spec/manifest-v2-2/#manifest-list

.. graphviz:: images/arrangement-v4.dot
   :caption: Orchestrator and worker builds (arrangement v4)

Arrangement version 5 (ODCS Integration)
----------------------------------------

This arrangement builds on version 4. The ``resolve_composes`` plugin
enables integration with `odcs`_. See `ODCS Integration`_ for details.

.. _`odcs`: https://pagure.io/odcs
.. _`ODCS Integration`: odcs.html

- worker build

  * No changes

- orchestrator build

  * resolve_composes
