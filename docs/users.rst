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
