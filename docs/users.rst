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

Arrangement version 2 (orchestrator creates filesystem)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

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
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This arrangement builds on version 2. The ``koji_promote`` plugin,
which was previously responsible for creating the Koji build, is
replaced by these plugins:

- worker build

  * koji_upload

- orchestrator build

  * fetch_worker_metadata

  * koji_import

  * koji_tag_build

Additionally the ``sendmail`` plugin now runs in the orchestrator
build and not the worker build.

Arrangement version 4 (Multiple worker builds)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This arrangement moves most of the Pulp integration work to the
orchestrator build, allowing for multiple worker builds. Only the
pulp_push plugin remains in the worker build.

A new plugin, group_manifests, creates a manifest list object in the
registry, grouping together the image manifests from the worker
builds.

.. graphviz:: images/arrangement-v4.dot
   :caption: Orchestrator and worker builds (arrangement v4)

Using Artifacts from Koji
-------------------------

During a container build, it might be desireable to fetch some artifacts
from an existing Koji build. For instance, when building a Java-based
container, JAR archives from a Koji build are required to be added to
the resulting container image.


The atomic-reactor pre-build plugin, fetch_maven_artifacts, can be used
for including non-RPM content in a container image during build time.
This plugin will look for the existence of two files in the git repository
in the same directory as the Dockerfile:
fetch-artifacts-koji.yaml and fetch-artifacts-url.yaml.

The first is meant to fetch artifacts from an existing Koji build.
The second allows specific URLs to be used for fetching artifacts.
fetch-artifacts-koji.yaml will be processed first.

fetch-artifacts-koji.yaml
~~~~~~~~~~~~~~~~~~~~~~~~~

::

  - nvr: foobar # All archives will be downloaded

  - nvr: com.sun.xml.bind.mvn-jaxb-parent-2.2.11.redhat_4-1
    archives:
    # pull a specific archive
    - filename: jaxb-core-2.2.11.redhat-4.jar
      group_id: org.glassfish.jaxb

    # group_id omitted - multiple archives may be downloaded
    - filename: jaxb-jxc-2.2.11.redhat-4.jar

    # glob support
    - filename: txw2-2.2.11.redhat-4-*.jar

    # pull all archives for a specific group
    - group_id: org.glassfish.jaxb

    # glob support with group_id restriction
    - filename: txw2-2.2.11.redhat-4-*.jar
      group_id: org.glassfish.jaxb

    # causes build failure due to unmatched archive
    - filename: archive-filename-with-a-typo.jar

Each archive will be downloaded to artifacts/<mavenfile_path> at the root
of git repository. It can be used from Dockerfile via ADD/COPY instruction:

::

  COPY \
    artifacts/org/glassfish/jaxb/jaxb-core/2.2.11.redhat-4/jaxb-core-2.2.11.redhat-4.jar /jars

The directory structure under ``artifacts`` directory is determined
by ``koji.PathInfo.mavenfile`` method. It’s essentially the end of
the URL after ``/maven/`` when downloading archive from Koji Web UI.

Upon downloading each file, the plugin will verify the file checksum by
leveraging the checksum value in the archive info stored in Koji. If
checksum fails, container build fails immediately. The checksum algorithm
used is dictated Koji via the `checksum_type` vaue of archive info.

If build specified in nvr attribute does not exist, the container
build will fail.

If any of the archives does not produce a match, the container build will fail.
In other words, every item in the archives list is expected to match at least
one archive from specified Koji build. However, the build will not fail if it
matches multiple archives.

*Note that only archives of maven type are supported.* If in the nvr
supplied an archive item references a non maven artifact, the container
build will fail due to no archives matching request.


fetch-artifacts-url.yaml
~~~~~~~~~~~~~~~~~~~~~~~~

::

  - url: http://download.example.com/JBossDV/6.3.0/jboss-dv-6.3.0-teiid-jdbc.jar
    md5: e85807e42460b3bc22276e6808839013
  - url: http://download.example.com/JBossDV/6.3.0/jboss-dv-6.3.0-teiid-javadoc.jar
    # Use different hashing algorithm
    sha256: 3ba8a145a3b1381d668203cd73ed62d53ba8a145a3b1381d668203cd73ed62d5
    # Optionally, overwrite target name
    target: custom-dir/custom-name.jar

Each archive will be downloaded to artifacts/<target_path> at the root
of git repository. It can be used from Dockerfile via ADD/COPY instruction:

::

  COPY artifacts/jboss-dv-6.3.0-teiid-jdbc.jar /jars/
  COPY artifacts/custom-dir/custom-name.jar /jars/

By default, target_path is set to the filename from provided url. It can
be customized by providing a target. The target value can be either a
filename, archive.jar, or also include a path, my/path/archive.jar, for
easier archive management.

The md5, sha1, sha256 attributes specify the corresponding hash to be used
when verifying artifact was downloaded properly. At least one of them is
required. If more than one is defined, multiple hashes will be computed
and verified.


Koji Build Metadata Integration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In the future, a reference of each artifact fetched by OSBS will be
added to the koji build metadata once imported via content generator API.
The list of components for the container image in output list will
include the fetched artifacts in addition to the installed RPMs.
