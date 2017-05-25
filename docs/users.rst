Building Container Images
=========================

Building images using fedpkg
----------------------------

Building images using koji
--------------------------

Streamed build logs
~~~~~~~~~~~~~~~~~~~

When atomic-reactor in the orchestrator build runs its
`orchestrate_build` plugin and watches the builds, it will stream in
the logs from those builds and emit them as logs itself, with the
platform name as one of the fields. The extra fields for these worker
logs will be: platform, level.

Note that there will be a single Koji task with a single log output,
which will contain logs from multiple builds. When watching this using
``koji watch-logs <task id>`` the log output from each worker build
will be interleaved. To watch logs from a particular worker build
image owners can use ``koji watch-logs <task id> | grep -w x86_64``.


Configuring koji-containerbuild for koji CLI
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Building images using osbs-client
---------------------------------

Configuring osbs-client
~~~~~~~~~~~~~~~~~~~~~~~

Can Orchestrate
'''''''''''''''

The parameter ``can_orchestrate`` defaults to false. The API method
``create_orchestrator_build`` will fail unless ``can_orchestrate`` is
true for the chosen instance section.

.. _reactor_config_secret:

Reactor config secret
'''''''''''''''''''''

When ``reactor_config_secret`` is specified this is the name of a
Kubernetes secret holding :ref:`config.yaml`. A pre-build plugin will
be configured with the location this secret is mounted.

.. _client_config_secret:

Client config secret
''''''''''''''''''''

When ``client_config_secret`` is specified this is the name of a
Kubernetes secret holding ``osbs.conf`` for use by atomic-reactor when it
creates worker builds. The `orchestrate_build` plugin is told the
path to this.

.. _token_secrets:

Token secrets
'''''''''''''

When ``token_secrets`` is specified the specified secrets (space
separated) will be mounted in the OpenShift build. When ":" is used,
the secret will be mounted at the specified path, i.e. the format is::

  token_secrets = secret:path secret:path ...

This allows an ``osbs.conf`` file (from ``client_config_secret``) to
be constructed with a known value to use for ``token_file``.

Accessing built images
----------------------

Building a buildroot using atomic-reactor
-----------------------------------------

Using atomic-reactor plugins
----------------------------

Creating a build JSON
---------------------

Writing a Dockerfile
--------------------

.. _`build process`:

Understanding the Build Process
-------------------------------

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
build is called the "arrangement", and currently there is one defined
arrangement.

Arrangement version 1 (delegation)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

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
''''''''''''

Steps performed by the orchestrator build are:

- For layered images:

  * pull the parent image in order to inspect environment variables
    (this is to allow substitution to be performed in the "release"
    label)

- Supply a value for the "release" label if it is missing and Koji
  integration is enabled (this value is provided to the worker build)

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
''''''

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
