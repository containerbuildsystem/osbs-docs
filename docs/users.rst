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

.. _container.yaml:

Image configuration
-------------------

Some aspects of the container image build process are controlled by a
file in the git repository named ``container.yaml``. This file need
not be present, but if it is it must adhere to the `container.yaml
schema`_.

.. _`container.yaml schema`: https://github.com/projectatomic/atomic-reactor/blob/master/atomic_reactor/schemas/container.json

An example::

  platforms:
    # all these keys are optional

    only:
    - x86_64   # can be a list (as here) or a string (as below)
    - ppc64le
    - armhfp
    not: armhfp

platforms
~~~~~~~~~

Keys in this map relate to multi-platform builds. The full set of
platforms for which builds may be required will come initially from
the Koji build tag associated with the build target, or from the
``platforms`` parameter provided to the ``create_orchestrator_build``
API method when Koji is not used.

only
  list of platform names (or a single platform name as a string); this
  restricts the platforms to build for using set intersection

not
  list of platform names (or a single platform name as a string);
  this restricts the platforms to build for using set difference

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

.. _image-tags:

Image tags
----------

The output from atomic-reactor includes container images tagged into a
registry (or Pulp, if Pulp integration is enabled). In addition, when
multi-platform builds are enabled each set of images will be grouped
into a manifest list, which itself is tagged.

While the repository name is specified by the ``name`` label in the
Dockerfile, the tags used within the repository are:

- a unique tag including the timestamp (this tag is the only tag
  applied for scratch builds)
- ``latest``
- ``{version}`` (the ``version`` label)
- ``{version}-{release}`` (the ``version`` and ``release`` labels together)
- any additional tags configured in the git repository (named in the
  ``additional-tags`` file)

These tags are applied to the manifest list or, if multi-platform
image builds are not enabled (see :ref:`group_manifests
<group-manifests>`), to the sole image manifest resulting from the
build.

Override Parent Image
----------------------

The parent image used for building a layered image is determined by the ``FROM``
instruction in the Dockerfile by default. Users can override this behavior
by specifying a koji parent build via the ``koji_parent_build`` API parameter.
When given, the image reference in the provided koji parent build will be used as
the value of the FROM instruction. The same source registry restrictions
apply.

Additionally, the koji parent build must use the same container image repository
as the value of the FROM instruction in Dockerfile. For instance, if the
Dockerfile states ``FROM fedora:27``, the koji parent build has to be of a
container image that pushed to the ``fedora`` repository. The koji parent build
may refer to a ``fedora:26`` image, but using a koji parent build for an image
that was pushed to ``rsyslog`` will cause a build failure.

This behavior requires koji integration to be enabled in the OSBS environment.

Koji NVR
--------

When koji integration is enabled, every container image build requires a unique
Name-Version-Release, NVR. The Name and Version are extracted from the **name**
and **version** labels in Dockerfile. Users can also use the **release** label
to hard code the release value, although this requires a git commit for every
build to change the value. A better alternative is to leave off the **release**
label which causes OSBS to query koji for what the next release value should be.
This is done via koji's ``getNextRelease`` API method. In either case, the
release value can also be overridden by using the ``release`` API parameter.

Isolated Builds
---------------

When a build is created via OSBS, the built container image is pushed to the
container registry updating various tag references in the container registry.
Some of these tag references are unique, while others are meant to transition
over time to reference a newer image, for instance ``latest`` and ``{version}``
tags. In other words, building a container image usually updates these
transient, non-unique, tags in container registry. In some cases, this is not
desired.

Consider the case of a container image that includes one, or more, packages that
have recently been identified as containing security vulnerabilities. To address
this issue, a new container image must be built. The difference for this build
is that only changes related to the security fix must be applied. Any unrelated
development that has occurred should be ignored. It would not be correct to
update the ``latest`` tag reference with this build.  To achieve this, the
concept of isolated builds was introduced.

As an example, let's use the image ``rsyslog`` again. At some point the
container image 7.4-2 is released (version 7.4, release 2). Soon after, minor
bug fixes are addressed in 7.4-3, a new feature is added to 7.4-4, and so on. A
security vulnerability is then discovered in the released image 7.4-2. To
minimize disruption to users, you may want to build a patched version of 7.4-2,
say 7.4-2.1. The packages installed in this new container image will differ from
the former only when needed to address the security vulnerability. It will not
include the minor bug fixes from 7.4-3, nor the new features added in 7.4-4. For
this reason, updating the ``latest`` tag is considered incorrect.

::

    7.4 version
    |
    |____
    |   |1 release
    |
    |__________________
    |   |2 release    |2.1 release
    |
    |____
    |   |3 release
    |
    |____
    |   |4 release
    |

To start an isolated build, use the ``isolated`` boolean parameter. Due to the
nature of isolated builds, the release value must be set via the ``release``
parameter which must match the format ``^\d+\.\d+(\..+)?$``

Isolated builds will only update the ``{version}-{release}`` unique tag and the
primary tag in target container registry.
