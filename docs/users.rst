Building Container Images
=========================

Building images using fedpkg
----------------------------

Following command submits a build to Koji::

    fedpkg container-build --target=<target>

For detailed Fedora workflow please visit `Fedora Layered Image Build System`_ guide.

.. _`Fedora Layered Image Build System`: https://docs.pagure.org/releng/layered_image_build_service.html



Building images using koji
--------------------------

Using a koji client CLI directly you have to specify git repo URL and branch::

    koji container-build <target> <repourl>#<branch/ref> --git-branch <branch>

The `koji-containerbuild plugin`_ provides the ``container-build`` sub-command
in the ``koji`` CLI. Please install the plugin in order to access this
sub-command::

    sudo yum install python3-koji-containerbuild-cli

You will now have the ``container-build`` sub-command available on your
workstation. For a full list of options::

    koji container-build --help

.. _`koji-containerbuild plugin`: https://github.com/containerbuildsystem/koji-containerbuild


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


Koji Build Results
~~~~~~~~~~~~~~~~~~

Koji Web
********

This is the easiest way to access information about OSBS builds.

List Builds
+++++++++++

Navigate to the "Builds" tab in koji and set the "Type" filter to ``image``.


Get Build
+++++++++

If you have the build ID, go to <KOJI_WEB_URL>/buildinfo?buildID=<build-ID>

If you want to search build by its name or part of name, use the search box on
top of the page.
For example, ``redis-*`` and select "Builds".


In koji build you can find a lot information about build, some noticeable are:

* pull specifications for the build in the 'Extra' section ``image.index.pull``,
  for digest type in ``image.index.digests``

* list of image archives for each specific architecture for which build was
  executed (for more detailed information about specific archive click on 'info')


Also in the "Extra" section, docker.config shows (parts of)
the `docker image JSON description`_, as well it indicates container image
API version.


See atomic-reactor documentation for
a `full description of the Koji container image metadata`_.

.. _`docker image JSON description`: https://github.com/moby/moby/blob/master/image/spec/v1.2.md#image-json-description
.. _`full description of the Koji container image metadata`: https://github.com/containerbuildsystem/atomic-reactor/blob/master/docs/koji.md#type-specific-build-metadata


Get Task
++++++++

All OSBS builds triggered via koji have a task linked to them. On the Build
info page, look at the "Extra" field for the ``container_koji_task_id`` value.
When you locate this task ID integer, go to
<KOJI_WEB_URL>/taskinfo?taskID=<task-ID> to find the task responsible
for the build.


Build Logs
++++++++++

The logs can be found in task's "Output" section (older builds will have that
section empty as logs has been garbage collected), or in build's "Logs" section
(persist after garbage collection).


Koji CLI
********

List Builds
+++++++++++

List all image (OSBS) builds::

    koji call listBuilds type=image

Apply filter for more specific search::

    koji call listBuilds type=image createdAfter='2016-02-01 00:00:00' prefix=redis

Search for builds of specific users::

    koji call listUsers prefix=<user>  # get user-ID
    koji call listbuilds type=image userId=<user-ID>


Get Build
+++++++++

Retrieve build information from either the build ID or the build NVR::

     koji buildinfo <build-ID or build-NVR>


Get Task
++++++++

The "Extra" field in build result is useful to track the task that originated
this build. Use the "container_koji_task_id", or "filesystem_koji_task_id",
to get more info about task::

    koji taskinfo <task-ID>


Cancel Task
+++++++++++

You can cancel a buildContainer koji task as for other types of task, and this
will cancel the OSBS build::

    koji cancel <task-ID>


Build Notifications
*******************

Package owners and build submitter will be notified via email about build.


Building images using osbs-client
---------------------------------

_`osbs-client` provides ``osbs`` CLI command for interaction with OSBS builds
and allows creation of new builds directly without koji-client.

Please note that mainly ``koji`` and ``fedpkg`` commands should be used
for building container images instead of direct ``osbs-client`` calls.

To execute build via osbs-client CLI use::

    osbs build -g <git_repo_url> -b <branch> -u <username> --git-commit <commit> [--platforms=x86_64] [-i <instance>]

To see full list of options execute::

    osbs build --help

To see all osbs-client subcommands execute::

    osbs --help

Please note that ``osbs-client`` must be configured properly using config file ``/etc/osbs.conf``.
Please refer to :ref:`osbs-client configuration section <configuring-osbs-client>`
for configuration examples.


.. `osbs-client`_: https://github.com/containerbuildsystem/osbs-client

Accessing built images
----------------------

Information about registry and image name is included in koji build. Use one
of names listed in ``extra.image.index.pull`` to pull built image from a registry.


Writing a Dockerfile
--------------------

A Dockerfile is required for building container images in OSBS. It must be
placed at the root of git repository. There can only be a single Dockerfile per
git repository branch.

Some labels are required to be defined:

- ``com.redhat.component``: the value of this label is used when importing a
  build into Koji via content generator API. We recommend that all images use
  a component string ending in ``-container`` here, so that you can easily
  distinguish these container builds from other non-container builds in Koji.
  The value can't be empty.
- ``name``: value is used to define the repository name in container registry to
  push built image. Limit this to lowercase alphanumerical values with the
  possibility to use dash as a word separator. A single ``/`` is also allowed.
  ``.`` is not allowed in the first section. For instance, **fed/rsys.log** and
  **rsyslog** are allowed, but **fe.d/rsyslog** and **rsys.log** aren't.
  The value can't be empty.
- ``version``: used as the version portion of Koji build NVR, as well as, for
  the version tag in container repository.
  The value can't be empty, and may be defined via ENV variable from parent.

For example::

    LABEL com.redhat.component=rsyslog-container \
          name=fedora/rsyslog \
          version=32

When OSBS builds a container image that defines the above labels, a Koji build
will be created in the format rsyslog-container-32-X. Where X is the release
value.  The container image will be available in container registry at:
``my-container-registry.example.com/fedora/rsyslog:32``.

The ``release`` label can also be used to specify the release value use for Koji
build. The value can't be empty, and may be defined via ENV variable from parent.
When omitted, the release value will be automatically determined by
querying Koji's getNextRelease API method.

Other labels are set automatically when not set in the Dockerfile:

- ``build-date``: Date/Time image was built as RFC 3339 date-time.
- ``architecture``: Architecture for the image.
- ``com.redhat.build-host``: OpenShift node where image was built.
- ``vcs-ref``: A reference within the version control repository; e.g. a git commit.
- ``vcs-type``: The type of version control used by the container source. Currently, only git is supported.

Although it is also possible to automatically include the ``vcs-url`` label, the default set
of automatically included labels does not include the label.

Sites wanting to include the ``vcs-url`` label to the set should do so by using custom
``orchestrator_inner:n.json`` and ``worker_inner:n.json`` specifying the full set of implicit labels
for the ``add_labels_in_dockerfile`` plugin::

    {
      "args": {
        "auto_labels": ["build-date", "architecture", "vcs-type", "vcs-url", "vcs-ref", "com.redhat.build-host"]
      },
      "name": "add_labels_in_dockerfile"
    },

Finally, it is also possible to set additional labels through the reactor
configuration, by setting the label key values in ``image_labels``.


.. _image-configuration:

Image configuration
-------------------

Some aspects of the container image build process are controlled by a
file in the git repository named ``container.yaml``. This file need
not be present, but if it is it must adhere to the `container.yaml
schema`_.

.. _`container.yaml schema`: https://github.com/containerbuildsystem/osbs-client/blob/master/osbs/schemas/container.json

An example::

  ---
  platforms:
    # all these keys are optional

    only:
    - x86_64   # can be a list (as here) or a string (as below)
    - ppc64le
    - armhfp
    not: armhfp

  go:
    modules:
      - module: example.com/go/packagename
      - module: example.com/go/anotherpackage
        archive: anotherpackage.tar.gz
        path: anotherpackage-v0.54.1

  compose:
    # used for requesting ODCS compose of type "tag"
    packages:
    - nss_wrapper  # package name, not an NVR.
    - httpd
    - httpd-devel
    # used for requesting ODCS compose of type "pulp"
    pulp_repos: true
    # used for requesting ODCS compose of type "module"
    modules:
    - "module_name1:stream1"
    - "module_name2:stream1"
    # Possible values, and default, are configured in OSBS environment.
    signing_intent: release
    # used for inheritance of yum repos and ODCS composes from baseimage build
    inherit: true

  image_build_method: docker_api

  autorebuild:
    from_latest: false
    add_timestamp_to_release: false

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

go
~~

Keys in this map relate to source code in the Go language which the
user intends to be built into the container image. They are
responsible for building the source code into an executable
themselves. Keys here are only for identifying source code which was
used to create the files in the container image.

modules
  sequence of mappings containing information for the Go modules (packages) built and shipped in the container image. The accepted mappings are listed bellow.

  module
    top-level go module (package) name to be built in the image. If ``modules`` is specified, this entry is required.

  archive
    possibly-compressed archive containing full source code including vendored dependencies.

  path
    path to directory containing source code (or its parent), possibly within archive.

.. _container.yaml-compose:

compose
~~~~~~~

This section is used for requesting yum repositories at build time. When this
section is defined, a compose will be requested by using ODCS.

packages
  list of package names to be included in ODCS compose. Package in this case
  refers to the "name" portion of the NVR (name-version-release) of an RPM, not
  the Koji package name. Packages will be selected based on the Koji build tag
  of the Koji build target used. The following command is useful in determining
  which packages are available in a given Koji build tag:
  ``koji list-tagged --inherit --latest TAG``

  If "packages" key is declared but is empty (``packages: []`` in YAML), the
  compose will include all packages from the Koji build tag of the Koji build
  target.

  ODCS will work more quickly if you only specify the minimum set of packages
  you need here, but if you want to avoid hard-coding a complete package list
  in ``container.yaml``, you can use the empty list to just make everything
  available.

pulp_repos
  boolean to control whether or not an ODCS compose of type "pulp" should be
  requested. If set to true, ``content_sets.yml`` must also be provided. A
  compose will be requested for each architecture in ``content_sets.yml``.
  See :ref:`content_sets.yml`.

modules
  list of modules for requesting ODCS compose of type "module". ODCS will
  cherry-pick each module into the compose.

  Use this ``modules`` option to make module builds available that are not yet
  available from the other options like Pulp. This is useful if you want to
  test a newly-built module before it is available in Pulp, or if you want to
  pin to a specific module that MBS has built.

  This list can be of the format ``name:stream``, ``name:stream:version``, or
  ``name:stream:version:context``.

  If you specify a ``name:stream`` without specifying a ``version:context``,
  ODCS will query MBS to find the very latest ``version:context`` build. For
  example, if you specify ``go-toolset:rhel8``, ODCS will query MBS for the
  latest ``go-toolset`` module build for the ``rhel8`` stream, whereas if you
  specify ``go-toolset:rhel8:8020020200128163444:0ab52eed``, ODCS will compose
  that exact module instead.

  Note that if you simply specify a ``name:stream`` for a module, ODCS will
  compose the very latest module that a module developer has built for that
  stream, and this module might not be tested by QE or GPG signed.
  Alternatively, if your desired module is already QE'd, signed, and available
  in Pulp, the ``pulp_repos: true`` option will ensure that your container
  build environment only uses tested and signed modules.

signing_intent
  used for verifying packages in yum repositories are signed with expected
  signing keys. The possible values for signing intent are defined in OSBS
  environment. See :ref:`config.yaml-odcs` section for environment configuration
  details, and full explanation of :ref:`signing-intent`.

inherit
  boolean to control whether or not to inherit yum repositories and odcs composes
  from baseimage build, default false. Scratch and isolated builds do not support
  inheritance and false is always assumed.

include_unpublished_pulp_repos
  If you set ``include_unpublished_pulp_repos: true`` under the ``compose``
  section in ``container.yaml``, the ODCS composes can pull from unpublished
  pulp repositories. The default is ``false``. Use this setting to make
  pre-release RPMs available to your container images. Use caution with this
  setting, because you could end up publicly shipping container images with
  RPMs that you have not exposed publicly otherwise.

ignore_absent_pulp_repos
  If you set ``ignore_absent_pulp_repos: true`` under the ``compose`` section
  in ``container.yaml``, ODCS will ignore missing content sets. Use this
  setting if you want to pre-configure your container's ``content_sets.yml``
  in dist-git before a Pulp administrator creates all the repositories you
  expect to use in the future. Alternatively, do not enable this setting if
  you want to enforce strict error-checking on all the the content set names
  in ``content_sets.yml``.

multilib_method
  List of methods used to determine if a package should be considered multilib.
  Available methods are ``iso``, ``runtime``, ``devel``, and ``all``.

multilib_arches
  Platform list for which the multilib should be enabled. For each entry in the
  list, ODCS will also include packages from other compatible architectures in
  the compose. For example when "x86_64" is included, ODCS will also include
  "i686" packages in the compose.

modular_koji_tags
  List of Koji tags in which the modular Koji Content Generator builds are
  tagged. Such builds will be included in the compose.

**If there is a "modules" key, it
must have a non-empty list of modules. The "packages" key, and only the "packages"
key, can have an empty list.**

**The "packages", "modules" and "pulp_repos" keys can be used mutually.**

flatpak
~~~~~~~

This section holds the information needed to build a Flatpak. For more
information on Flatpak builds, see `flatpak-docs`_.
This is a map with the following keys:

id
  The ID of the application or runtime. Required.

name
  ``name`` label in generated Dockerfile. Used for the repository when pushing
  to a registry. Defaults to the module name.

component
  ``com.redhat.component`` label in generated Dockerfile. Used to name the
  build when uploading to Koji. Defaults to the module name.

base_image
  The image that is used when installing packages to create the filesystem.
  It is also recorded as the parent image of the output image. This
  defaults to the ``flatpak: base_image`` setting in the **reactor-config-map**.

branch
  The branch of the application or runtime. In many cases, this will match the
  stream name of the module. Required.

cleanup-commands
  A shell script that is run after installing all packages. Only applicable to
  runtimes.

command
  The name of the executable to run to start the application. If not specified,
  defaults to the first executable found in /usr/bin. Only applicable to
  applications.

tags
  Tags to add to the Flatpak metadata for searching. Only applicable to
  applications.

finish-args
  Arguments to ``flatpak build-finish`` (see the flatpak-build-finish man page).
  This is a string split on white space with shell style quoting. Only
  applicable to applications.

.. _`flatpak-docs`: https://github.com/containerbuildsystem/atomic-reactor/blob/master/docs/flatpak.md

tags
~~~~

List of tags to be applied to the built image. When this option is specified,
the tags described will be applied to the image. If present, the ``{version}``,
``latest``, and the tags listed in the ``additional-tags`` file will no longer
be automatically applied. See the `image-tags`_ section below for further
reference.

version
~~~~~~~

This key is no longer used by OSBS and is only kept in the schema for backwards
compatibility.


set_release_env
~~~~~~~~~~~~~~~

Optional string.  If set, osbs-client will modify each stage of the image's 
Dockerfile, adding an ENV statement immediately following the FROM statement.
The ENV statement will assign an environment variable with the same name as
the value of set_release_env and the value of the current build's release number.
Users can use this environment variable to get the release value when running
tools inside the container.

.. _container.yaml-autorebuild:

autorebuild
~~~~~~~~~~~

This map accepts keys, as described below. This values are only used
for autorebuilds, if autorebuilds are enabled.

from_latest
  Boolean to control whether to rebuild from the latest commit in the build
  branch. Defaults to ``false``.

add_timestamp_to_release
  Boolean to control whether to append timestamp to explicitly specified release for autorebuilds.
  Defaults to ``false``. When ``true`` it will append timestamp to release with ``.`` separator.
  For example if name is ``fedora/rsyslog``, version is ``32``, and release is ``5``,
  the container image will be available in container registry at:
  ``my-container-registry.example.com/fedora/rsyslog:32-5.20191007151825``.

ignore_isolated_builds
  Boolean to control whether to rebuild when parent image triggering build was isolated build.

.. _osbs-config-autorebuild:

Automatic Rebuilds
~~~~~~~~~~~~~~~~~~

This section specifies whether and how a build should be rebuilt based on
changes to the base parent image.

By default autorebuild is disabled. The feature can be enabled by making some
changes in your dist-git repo and submitting a container build.


Enabling Automatic Rebuilds
***************************

Enable autorebuild in config::

    fedpkg container-build-setup --set-autorebuild true

This will create/update the .osbs-repo-config file.
The file will be automatically added for commit.

Finally, add all modified files, commit, and push modifications.
For these **changes to take place, request a container build** as usual::

    fedpkg container-build

This must be a regular non-scratch/non-isolated build.
The steps above apply to a single branch in your dist-git repo.
It must be repeated for each branch you wish to enable the feature.

The next time the parent image used by your container image is updated, your image will be automatically rebuilt.


Disabling Automatic Rebuilds
****************************

First, use fedpkg to disable autorebuild::

    fedpkg container-build-setup --set-autorebuild false

This will create/update the .osbs-repo-config file.
The file will be automatically added for commit.

Finally, commit and push modifications. For these **changes to take place,
request a container build** as usual::

    fedpkg container-build

This must be a regular non-scratch/non-isolated build.
The steps above apply to a single branch in your dist-git repo.
It must be repeated for each branch you wish to disable the feature.


image_build_method
~~~~~~~~~~~~~~~~~~

This string indicates which build-step plugin to use in order to perform the
layered image build, on a per-image basis. The **docker_api** plugin uses
the docker-py module to run the build via the Docker API, while the
**imagebuilder** plugin uses the imagebuilder_ utility to do the same.
Both have similar capabilities, but the **imagebuilder** plugin brings two
advantages:

1. It performs all changes made in the build in a single layer, which is
   a little more efficient and removes the need to squash layers afterward.
2. It can perform multistage builds without requiring Docker 17+ (which
   Red Hat and Fedora do not support).

In order to use the **imagebuilder** plugin, the imagebuilder_ binary must be
available and in the PATH for the builder image, or an error will result.

.. _imagebuilder: https://github.com/openshift/imagebuilder/

.. _cachito-usage:

Fetching source code from external source using cachito
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

As described in :ref:`cachito-integration`, it is possible to use cachito to
download a tarball with an upstream project and its dependencies and make it
available for usage during an OSBS build.

remote_source
~~~~~~~~~~~~~

This map contains configuration of what sources OSBS will request from cachito
and how they will be requested. The keys accepted here are described below. If
OSBS cachito integration is not configured in the OSBS instance, the entries
here will be ignored.

repo
  String with an URL to the upstream project SCM repository, such as
  ``https://git.example.com/team/repo.git``. Required.

ref
  String with a 40-character reference to the SCM reference of re project
  described in ``repo`` to be fetched. This should be a complete git commit
  hash. Required.

pkg_managers
  A list of package managers to be used for resolving the upstream project
  dependencies. If not provided, cachito will try to figure out what manager to
  use here.

flags
  List of flags to pass to the cachito request. See the cachito_ documentation
  for further reference.

Once the `remote_source` map described above is set in ``container.yaml``, you
can now copy the upstream sources (with bundled dependencies) provided by
cachito in your build image by adding::

    COPY $REMOTE_SOURCE $REMOTE_SOURCE_DIR

to your Dockerfile. From this point on, you will find the root of the upstream
project at ``$REMOTE_SOURCE_DIR/app``, and the project dependencies at
``$REMOTE_SOURCE_DIR/deps``.

Note that ``$REMOTE_SOURCES_DIR`` is a build arg, available only in build time,
with the absolute path to the directory where the sources are expected to be
(OSBS sets other build args such as ``GOPATH`` for Golang sources relying on
the existence of the dependencies in this directory). Hence, for cleaning up
the image after using the sources, add the following line to the Dockerfile
after the build is complete::

    RUN rm -rf $REMOTE_SOURCE_DIR

``$REMOTE_SOURCE`` is another build arg, which points to the extracted tar
archive provided by cachito in the buildroot workdir.

.. note:: To better use the cachito provided dependencies, a full gomod
  supporting Golang version is required. In other words, you should use Golang
  >= 1.13

Replacing project dependencies with cachito
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Cachito also provides a feature to allow users to replace a project's
dependencies with another version of that same dependency or with a completely
different dependency (this is useful when you want to use a patched fork for a
dependency).

OSBS allows users to use this feature for test purposes. In other words, you
can use cachito dependency replacements for scratch builds, and
**only for scratch builds**.

You can use this feature using the ``--replace-dependency`` option, which is
available for the ``fedpkg``, ``koji``, and ``osbs`` commands.

This option expects a string with the following information, separated by the
``:`` character: ``pkg_manager:name:version[:new_name]``, where ``pkg_manager``
is the package manager used by cachito to handle the dependency; ``name`` is
the name of the dependency to be replaced; ``version`` is the new version of
the dependency to be injected by cachito; and ``new_name`` is an optional
entry, to inform cachito that the dependency known as ``name`` by the package
manager should be replaced with a new dependency, known as ``new_name`` by the
package manager.::

  fedpkg container-build --scratch --replace-dependency gomod:pagure.org/cool-go-project:v1.2 gomod:gopkg.in/foo:2:github.com/bar/foo

or::

  koji container-build [...] --scratch --replace-dependency gomod:pagure.org/cool-go-project:v1.2 --replace-dependency gomod:gopkg.in/foo:2:github.com/bar/foo

In the examples above, two dependencies would be replaced. cool-go-project
would be used in version ``v1.2``, no matter what version is specified by the
project requesting it. Whereas ``gopkg.in/foo`` will be replaced by
``github.com/bar/foo`` version 2.

Note that while in ``fedpkg`` the replace dependency option receives multiple
parameters, the same option should be specified multiple times in ``koji`` or
the ``osbs`` CLI. This was done to keep the consistency with the similar option
to specify yum repository URLs in each particular CLI.

.. _cachito: https://github.com/release-engineering/cachito

.. _content_sets.yml:

Content Sets
------------

The file ``content_sets.yml`` is used to define the content sets relevant to the
container image.  This is relevant if RPM packages in container image are in
pulp repositories. See ``pulp_repos`` in :ref:`container.yaml-compose` for how
this file is used during build time. If this file is present, it must adhere
to the `content_sets.yml schema`_.

.. _`content_sets.yml schema`: https://github.com/containerbuildsystem/atomic-reactor/blob/master/atomic_reactor/schemas/content_sets.json

An example::

  ---
  x86_64:
  - server-rpms
  - server-extras-rpms

  ppc64le:
  - server-for-power-le-rpms
  - server-extras-for-power-le-rpms

OSBS will create a /root/buildinfo/content_manifests/`{IMAGE_NVR}.json` file in
the final built image containing platform specific content sets information.

For instance::

    {
      "metadata": {
        "image_layer_index": 3
      },
      "content_sets": ["server-rpms", "server-extras-rpms"]
    }

where `image_layer_index` is the index for the most recent layer for that
image.

Using Artifacts from Koji
-------------------------

During a container build, it might be desirable to fetch some artifacts
from an existing Koji build. For instance, when building a Java-based
container, JAR archives from a Koji build are required to be added to
the resulting container image.


The atomic-reactor pre-build plugin, fetch_maven_artifacts, can be used
for including non-RPM content in a container image during build time.
This plugin will look for the existence of two files in the git repository
in the same directory as the Dockerfile:
fetch-artifacts-koji.yaml and fetch-artifacts-url.yaml.  (See `fetch-artifacts-url.json`_ and `fetch-artifacts-nvr.json`_ for their YAML schema.)

.. _`fetch-artifacts-url.json`: https://github.com/containerbuildsystem/atomic-reactor/blob/master/atomic_reactor/schemas/fetch-artifacts-url.json

.. _`fetch-artifacts-nvr.json`: https://github.com/containerbuildsystem/atomic-reactor/blob/master/atomic_reactor/schemas/fetch-artifacts-nvr.json

The first is meant to fetch artifacts from an existing Koji build.  The second
allows specific URLs to be used for fetching artifacts.  Note that all
combinations of the yaml files here described are valid, i.e., you can have
either one of the two files in your repository or you could have both files in
the repository. ``fetch-artifacts-koji.yaml`` will be processed first.

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
used is dictated by Koji via the `checksum_type` value in the archive info.

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

OSBS's atomic-reactor pushes the new container image to the container
registry (or Pulp, if Pulp integration is enabled) and updates various tag
references in the registry. In addition, when multi-platform builds are
enabled, atomic-reactor groups each set of images into a manifest list
and tags that manifest list.

OSBS determines the name of the repository from the ``name`` label in the
Dockerfile. There are three categories of tags that OSBS creates when
tagging the resulting image in the registry:

* A "**unique**" tag: This tag includes the timestamp of when the image
  was built.  For scratch builds, this is the only tag that OSBS applies.
  Example: ``rsync-containers-candidate-93619-20191017205627``

* A "**primary**" tag: This tag is the ``{version}-{release}`` for the
  image (a combination of the ``version`` and ``release`` labels in the
  Dockerfile). Example: ``4-2``. This tag is unique for each Koji build.

* "**floating**" tag(s): These tags transition to newer image references
  over time. In other words, every time you build a new container image,
  OSBS updates these floating tags. Examples: ``latest``, or ``{version}``

  Floating tags are configurable. If you set ``tags`` in container.yaml, OSBS
  applies those tags to your newly-built image as floating tags.

  If you do not set ``tags`` in container.yaml, OSBS applies the following
  floating tags automatically:

  - ``{version}`` (the ``version`` label)
  - ``latest``
  - any additional tags named in the ``additional-tags`` file (DEPRECATED
    and will no longer be supported in a future version. Please consider using
    ``tags`` in container.yaml instead)

These tags are applied to the manifest list or, if multi-platform
image builds are not enabled (see :ref:`group_manifests
<group-manifests>`), to the sole image manifest resulting from the
build.

Override Parent Image
----------------------

OSBS uses the ``FROM`` instruction in the Dockerfile to find a parent image
for a layered image build. Users can override this behavior by specifying a
koji parent build via the ``koji_parent_build`` API parameter. When a user
specifies a ``koji_parent_build`` parameter, OSBS will look up the image
reference for that koji build and override the ``FROM`` instruction with that
image instead. The same source registry restrictions apply. For multi-stage
builds, the ``koji_parent_build`` parameter will only override the final
``FROM`` instruction.

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

During the build process, OSBS will query koji for the builds of all parent
images using their NVRs. If any of the parent image builds is not found in
koji, or if NVR information cannot be extracted from the parent image, OSBS
assumes that the parent image was not built by OSBS and halts the current
build. In other words, an image cannot be built using a parent image which has
not been built by OSBS. It is possible to disable this feature through reactor
configuration, with ``skip_koji_check_for_base_image`` option in
`config.json`_, when there are no NVR labels set on the base image,
if the NVR labels are set on the base image, the check is performed regardless.

Digests verification
~~~~~~~~~~~~~~~~~~~~

Once OSBS has the koji build information for a parent image, it compares the
digest of the parent image manifest available in koji metadata (stored when
that parent build had completed) with the actual parent image manifest digest
(calculated by OSBS during the build). In case manifests do not match, the build
will fail and the parent image **must** be rebuilt in OSBS before it is used in
another build.

If the manifest in question is a manifest list and the digests comparison fail,
the V2 manifest digests in the manifest list will be compared with the koji
build archive metadata digests. In this case, OSBS will only halt the build
with an error, advising rebuilding the parent image, if the V2 manifest digests
in the manifest list do not match the analogous koji information.  This
behavior can be deactivated through the ``deep_manifest_list_inspection``
option. See `config.json`_ for further reference.

Manifest lists can be manually pushed to the registry to make sure a specific tag
(e.g., latest) is available for all platforms. In such cases, these manifest lists
may include images from different koji builds. OSBS will only perform digest checks
for the images requested in the current build. Moreover, build requests for platforms
that were not built in the same koji build as the one found for the given image
reference (manifest list) will fail.

It is also possible to have OSBS only warn about any digest mismatches (instead
of halting the build with an error). This is done by setting the
``fail_on_digest_mismatch`` option to false in the `config.json`_ file.

Isolated Builds
---------------

In some cases, you may not want to update the floating tags for certain
builds.

Consider the case of a container image that includes packages that have new
security vulnerabilities. To address this issue, you must build a new
container image. You only want to apply changes related to the security fixes,
and you want to ignore any new unrelated development work. It is not correct
to update the ``latest`` floating tag reference for this build. You can use
OSBS's isolated builds feature to achieve this.

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
nature of isolated builds, you must explicitly specify your build's
``release`` parameter, which must match the format ``^\d+\.\d+(\..+)?$``.

Here is an example of an isolated build using fedpkg::

  fedpkg container-build --isolated --build-release=2.1

Isolated builds will only create the ``{version}-{release}`` primary tag and
the unique tag in the container registry. OSBS does not update any floating
tags for an isolated build.

Yum repositories
----------------

In most cases, part of the process of building container images is to install
RPM packages. These packages must come from yum repositories. There are various
methods for making a yum repository available for your container build.

.. _yum-repositories-odcs-compose:

ODCS compose
~~~~~~~~~~~~

The preferred method for injecting yum repositories in container builds is by
enabling ODCS integration via the "compose" key in ``container.yaml``. See
:ref:`image-configuration` and :ref:`signing-intent` for details.

RHEL subscription
~~~~~~~~~~~~~~~~~

If the underlying host is Red Hat Enterprise Linux (RHEL), its subscriptions
will be made available during container builds. Note that changes in the
underlying host to enable/disable yum repositories is not reflected in container
builds. ``Dockerfile`` must explicitly enable/disable yum repositories as
needed. Although this is desirable in most cases, in an OSBS deployment it can
cause unexpected behavior. It's recommended to disable subscription for RHEL
hosts when they are being used by OSBS.

Yum repository URL
~~~~~~~~~~~~~~~~~~

As part of a build request, you may provide the ``repo-url`` parameter with the
URL to a yum repository file. This file is injected into the container build.
Current OSBS versions support the combination of ODCS composes with repository files.
This is a change to OSBS former behavior, where the ODCS compose would be
disabled if a repository file URL was given.

Koji tag
~~~~~~~~

When Koji integration is enabled, a Koji build target parameter is provided. The
yum repository for the build tag of target is automatically injected in
container build. This behavior is disabled if either "ODCS compose" or "Yum
repository URL" are used.

Inherited yum repository and ODCS compose
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
If you want to inherit yum repositories and ODCS composes from baseimage build,
you can enable it via the "inherit" key under "compose" in ``container.yaml``.
Does not support scratch or isolated builds.
See :ref:`image-configuration`.

.. _signing-intent:

Signing intent
--------------

When the "compose" section in ``container.yaml`` is defined, ODCS composes will
be requested at build time. ODCS is aware of RPM package signatures and can be
used to ensure that only signed packages are added to the generated yum
repositories. Ultimately, this can be used to ensure a container image only
contains packages signed by known signing keys.

Signing intents are an abstraction for signing keys. It allows the OSBS
environment administrator to define which signing keys are valid for different
types of releases. See :ref:`config.yaml-odcs` section for details.

For instance, an environment may provide the following signing intents:
``release``, ``beta``, and ``unsigned``. Each one of those intents is then
mapped to a list of signing keys. These signing keys are then used during ODCS
compose creation. The packages to be included must have been signed by any of
the signing keys listed. In the example above, the intents could be mapped to
the following keys::

    # Only include packages that have been signed by "my-release-key"
    release -> my-release-key
    # Include packages that have been signed by either "my-beta-key" or
    # "my-release-key"
    beta -> my-beta-key, my-release-key
    # Do not check signature of packages - may include unsigned packages
    unsigned -> <empty>

The signing intents are also defined by their restrictive order, which will be
enforced when building layered images. For instance, consider the case of two
images, X and Y. Y uses X as its parent image (FROM X). If image X was built
with "beta" intent, image Y's intent can only be "beta" or "unsigned". If the
dist-git repo for image Y has it configured to use "release" intent, this value
will be downgraded to "beta" at build time.

Automatically downgrading the signing intent, instead of failing the build, is
important for allowing a hierarchy of layered images to be built automatically
by ``ImageChangeTriggers``. For instance, with Continuous Integration in mind, a
user may want to perform daily builds without necessarily requiring signed
packages, while periodically also producing builds with signed content. In this
case, the ``signing_intent`` in ``container.yaml`` can be set to ``release`` for
all the images in hierarchy. Whether or not the layered images in the hierarchy
use signed packages can be controlled by simply overriding the signing intent of
the top most ancestor image. The signing intent of the layered images would then
be automatically adjusted as needed.

In the case where multiple composes are used, the least restrictive intent is
used. Continuing with our previous signing intent example, let's say a container
image build request uses two composes. Compose 1 was generated with no signing
keys provided, and compose 2 was generated with "my-release-key". In this case,
the intent is "unsigned".

Compose IDs can be passed in to OSBS in a build request. If one or more compose
IDs are provided, OSBS will classify the intent of the existing compose. This
is done by inspecting the signing keys used for generating the compose and
performing a reverse mapping to determine the signing intent. If a match cannot
be determined, the build will fail. Note that if given compose is expired or
soon to be expired, OSBS will automatically renew it.

The ``signing_intent`` specified in ``container.yaml`` can be overridden with
the build parameter of same name. This particular parameter will be ignored for
autorebuilds. The value in ``container.yaml`` should always be used in that
case. Note that the signing intent used by the compose of parent image is still
taken into account which may lead to downgrading signing intent for the layered
image.

The Koji build metadata will contain a new key,
``build.extra.image.odcs.signing_intent_overridden``, to indicate whether or not
the ``signing_intent`` was overridden (CLI parameter, automatically downgraded,
etc). This value will only be ``true`` if
``build.extra.image.odcs.signing_intent`` does not match the ``signing_intent``
in ``container.yaml``.


Base image builds
-----------------

OSBS is able to create base images, and it does by creating Koji image-build task,
importing its output as a new container image, then continuing to build
using a Dockerfile that inherits from that imported image.

Each dist-git branch should have the following files:

* Dockerfile
* image-build.conf
* kickstart.ks (or any .ks name, but must match what image-build.conf references)

The Dockerfile should start "FROM koji/image-build", and continue with LABEL
and CMD etc instructions as needed.


The image-build.conf file should start "[image-build]" and set the target
(for the image-build task), distro, and ksversion, for example::

  [image-build]
  target = f30
  distro = Fedora-30
  ksversion = Fedora

The image-build task will need to know where to find the kickstart configuration; it finds this
from the 'ksurl' and 'kickstart' parameters in image-build.conf. If these are
absent from the file in dist-git, atomic-reactor will provide defaults:

* kickstart: 'kickstart.ks'
* ksurl: the dist-git URL and commit hash used for the OSBS build


In this way, the kickstart configuration can be placed in the dist-git
repository as 'kickstart.ks' alongside the Dockerfile and image-build.conf
files, and the correct git URL and commit hash will be recorded in Koji when
the image is built. This is the recommended way of providing a kickstart
configuration for base images.

Alternatively it can be stored elsewhere (perhaps another git repository) in
which case a URL is needed. However, when doing this please make sure to use
a git commit hash in the 'ksurl' parameter instead of a symbolic name
(e.g. branch name); failure to do this means there will be no reliable way
to discover the kickstart configuration used for the built image.


To execute base image build, run::

  fedpkg container-build --target=<target> --repo=url=<repo-url>

The --repo-url parameter specifies the URL to a repofile. The first section
of this is inspected and the 'baseurl' is examined to discover the compose URL.
You can also use --compose-id parameter to specify ODCS composes from which additional
yum repos will be used.


Multistage builds
-----------------

Often users may wish to build an image directly from project sources (rather
than intermediate build artifacts), but not include the sources or toolchain
necessary for compiling the project in the final image. Multistage builds are a
simple solution.

Multistage refers to container image builds with at least two stages in the
Dockerfile; initial stage(s) provide a build environment and produce some kind
of artifact(s) which in the final stage are copied into a clean base image. The
most obvious signature of a multistage build is that the Dockerfile has more
than one "FROM" statement. For example::

    FROM toolchain:latest AS builder1
    ADD .
    RUN make artifact

    FROM base:release
    COPY --from=builder1 artifact /dest/

In most respects, multistage builds operate very similarly to multiple
single-stage builds; the results from initial stage(s) are simply not tagged or
used except by later ``COPY --from`` statements. Refer to `Docker multistage
docs`_ for complete details.

.. _`Docker multistage docs`: https://docs.docker.com/develop/develop-images/multistage-build/

In OSBS, multistage builds require using the **imagebuilder** plugin, which
can be configured as the system default or per-image in ``container.yaml``.

In a multistage build, yum repositories are made available in all stages. The
build may have multiple parent builds, as each stage may specify a different
image. The parent images FROM initial stages are pulled and rewritten similarly
as the parent in the final stage (known as the "base image").  Note that ENV
and LABEL entries from earlier stages do not affect later stages.

Note that the ``COPY --from=<image>`` form (with a full image specification as
opposed to a stage alias) should not be used in OSBS builds. It works, but the
image used is not treated as other parents are (rewritten, etc). To achieve the
same effect, specify such images with another stage, for example::

    FROM registry.example.com/image:tag AS source1
    FROM base
    COPY --from=source1 src/ dest/


.. _operator-manifests:

Operator manifests
------------------

OSBS is able to extract operator_ manifests from an operator image. This image
should contain a ``/manifests`` directory, whose content can be extracted to
koji for later distribution.

.. _operator: https://coreos.com/operators/

To activate the operator manifests extraction from the image, you must set a
specific label in your Dockerfile to identify your build as either an operator
bundle build or an appregistry build::

    LABEL  com.redhat.delivery.appregistry=true
    LABEL  com.redhat.delivery.operator.bundle=true

Only one of these labels (the appropriate one for your build) may be present,
otherwise build will fail.

When present (and set to ``true``), this label triggers the atomic-reactor
``export_operator_manifests`` plugin. This plugin extracts the content from the
``/manifests`` directory in the built image and uploads it to koji. If the
``/manifests`` directory is either empty or not present in the image, the build
will fail.

Since the operator manifests are not tied to any specific architecture, OSBS
will decide from which worker build the manifests will be extract (and make
sure only a single platform will upload the archive to koji). If, for some
reason, you need to select which platform will extract and upload the manifests
archive, you can set the ``operator_manifests_extract_platform`` build param to
the desired platform.

[Backward compatibility] If the build succeeds,
the ``build.extra.operator_manifests_archive`` koji
metadata will be set to the name of the archive containing the operator
manifests (currently, ``operator_manifests.zip``).

The operator manifests archive is uploaded to koji as a separate type:
``operator-manifests`` (currently with filename ``operator_manifests.zip``).


.. _`config.json`: https://github.com/containerbuildsystem/atomic-reactor/blob/master/atomic_reactor/schemas/config.json

.. _operator-bundle:

Operator manifest bundle builds
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This type of build is for the newer style of operator manifests targeting
Openshift 4.4 or higher.
It is identified by the ``com.redhat.delivery.operator.bundle`` label.

To make OSBS cooperate on building your operator manifest bundle, you will need
to set up the following:

**Dockerfile**

.. code-block:: Dockerfile

    # Base needs to be scratch
    FROM scratch

    # Make this an operator bundle build
    LABEL com.redhat.delivery.operator.bundle=true

    # Does not matter where you keep your manifests in the repo, but in the
    # final image, they need to be in /manifests
    COPY my-manifests-dir/ /manifests

**container.yaml** (see ``operator_manifests`` in `container.yaml schema`_)

.. code-block:: yaml

    operator_manifests:
      # Relative path to your manifests dir from root of repo
      manifests_dir: my-manifests-dir

Pinning pullspecs for related images
************************************

In addition to extracting the ``/manifests`` dir to koji after build as
described above, the ``pin_operator_digest`` plugin is able to pin related
image pullspecs in your **ClusterServiceVersion** files.

Put simply, ``pin_operator_digest`` replaces floating tags in image pullspecs
with manifest list digests, creates a ``.spec.relatedImages`` section in the
file and puts all the pullspecs in it.

.. note:: If your **ClusterServiceVersion** file has a non-empty
          ``.spec.relatedImages`` section, OSBS will assume that it already
          contains correctly pinned pullspecs and will not touch the file.

Additionally, the plugin provides repo and registry replacement to allow
workflows where you use private or testing pullspecs in your manifest and OSBS
replaces them with final (perhaps customer-facing) pullspecs.

Replacing pullspecs
+++++++++++++++++++

Each step of pullspec replacement can be enabled/disabled using the
corresponding option in ``container.yaml``. By default, all steps are enabled.

.. code-block:: yaml

    operator_manifests:
      enable_digest_pinning: true
      enable_repo_replacements: true
      enable_registry_replacements: true

digest pinning
  Query registry for manifest list digest, replace tag with it.

  E.g. ``private.com/test/foobar:v1``
  -> ``private.com/test/foobar@sha256:123...``

repo replacements
  Query registry for package name (determined by component label in image),
  replace namespace/repo based on said package name. Only applies to images
  from registries that have a package mapping configured, either in OSBS site
  configuration or in ``container.yaml``:

  .. code-block:: yaml

    operator_manifests:
      repo_replacements:
        - registry: private.com
          package_mappings:
            foobar-package: foo/bar

  E.g. ``private.com/test/foobar@sha256:123...``
  -> ``private.com/foo/bar@sha256:132...``

  It may happen that the registry of one of your images has a package mapping,
  but is missing the replacement / has multiple possible replacements for
  a package. In this case, the build will fail and you will need to define the
  replacement in ``container.yaml`` as shown above.

registry replacements
  Based purely on OSBS site configuration, after digest is pinned and repo is
  replaced, registry may also be replaced if atomic-reactor is configured to
  do so.

  E.g. ``private.com/foo/bar@sha256:123...``
  -> ``public.io/foo/bar@sha256:123...``

Pullspec locations
++++++++++++++++++

Before OSBS can pin your pullspecs, it first needs to find them. Because
it is practically impossible to tell if a string is a pullspec, atomic-reactor
has a predefined set of locations where it will look for pullspecs.

1. metadata.annotations.containerImage anywhere in the file

   jq: ``.. | .metadata?.annotations.containerImage | select(. != null)``

2. All containers in each deployment

   jq: ``.spec.install.spec.deployments[].spec.template.spec.containers[]``

3. All initContainers in each deployment

   jq: ``.spec.install.spec.deployments[].spec.template.spec.initContainers[]``

4. All RELATED\_IMAGE\_* variables for all containers and initContainers

   jq: ``.env[] | select(.name | test("RELATED_IMAGE_"))`` for each of [2], [3]

5. All pullspecs from all annotations. This is done heuristically (OSBS needs
   to guess what might be a pullspec). See *heuristic annotations* below.

**Heuristic annotations**

OSBS will attempt to extract all pullspecs from all attributes of each
``metadata.annotations`` object in a CSV. If an attribute contains more than
one pullspec (as text, e.g. comma-separated), all of them should be found.
`Here <pullspec-heuristic_>`_ is how OSBS implements this. One important
thing to note is that only pullspecs conforming to a relatively strict format
will be found this way:

``registry/namespace*/repo:tag`` or ``registry/namespace*/repo@sha256:digest``

Any number of namespaces, including 0, is valid. Registry must contain at least
one dot: ``registry.io`` is valid but ``localhost`` is not. Digest, if present,
must be exactly 64 base16 characters. Tag or digest *must* be specified,
implicit ``latest`` tag is not supported.

.. _pullspec-heuristic: https://github.com/containerbuildsystem/atomic-reactor/commit/a8294d0e862e1465b95071af0e0722b2c21a8328

.. _example-csv:

**example.clusterserviceversion.yaml**

.. code-block:: yaml

    kind: ClusterServiceVersion
    metadata:
      annotations:
        containerImage: registry.io/namespace/foo  # [1]
        foobar: registry.io/foobar:latest, registry.io/ham/jam@sha256:... # [5]
    spec:
      install:
        spec:
          deployments:
          - spec:
              template:
                metadata:
                  annotations:
                    containerImage: registry.io/namespace/bar  # [1]
                spec:
                  containers:
                  - name: baz
                    image: registry.io/namespace/baz  # [2]
                    env:
                    - name: RELATED_IMAGE_SPAM
                      value: registry.io/namespace/spam  # [4]
                  initContainers:
                  - name: eggs
                    image: registry.io/namespace/eggs  # [3]

Creating the relatedImages section
++++++++++++++++++++++++++++++++++

Each entry in the ``.spec.relatedImages`` section needs a ``name`` and
an ``image`` - image being simply the pullspec itself. The name is constructed
differently depending on where the pullspec was found.

annotations :ref:`[1], [5] <example-csv>`
  Take repo name and tag/digest from pullspec, add "-annotation"

  E.g.

  - ``registry.io/foo:v1.1`` -> ``foo-v1.1-annotation``

  - ``registry.io/foo@sha256:123abc...`` -> ``foo-123abc...-annotation``

containers :ref:`[2] <example-csv>`
  Reuse original name, e.g. ``{name: baz, image: ...}`` -> ``baz``

initContainers :ref:`[3] <example-csv>`
  Same as containers

RELATED\_IMAGE env vars :ref:`[4] <example-csv>`
  Take name of variable, remove ``RELATED_IMAGE_`` prefix, convert to lowercase

  E.g. ``RELATED_IMAGE_SPAM`` -> ``spam``

The final ``relatedImages`` section of the example file would look like this:

.. code-block:: yaml

  spec:
    relatedImages:
    - name: foo-<digest>-annotation
      image: registry.io/namespace/foo@sha256:...
    - name: bar-<digest>-annotation
      image: registry.io/namespace/bar@sha256:...
    - name: baz
      image: registry.io/namespace/baz@sha256:...
    - name: eggs
      image: registry.io/namespace/eggs@sha256:...
    - name: spam
      image: registry.io/namespace/spam@sha256:...
    - name: jam-<digest>-annotation
      image: registry.io/ham/jam@sha256:...
    - name: foobar-<digest>-annotation
      image: registry.io/foobar@sha256:...

OSBS makes no guarantees about the order in which relatedImages will be added.

If 2 or more entries with the same name are found, then their images must also
match, otherwise this is a conflict and build will fail. This is especially
important for annotations, where the name is a combination of repo and tag.
Using 2 images with the same repo and tag but different registry/namespace
is not allowed.

Operator manifest appregistry builds
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This type of build is for the older style of operator manifests targeting
Openshift 4.3 or lower.
It is identified by the ``com.redhat.delivery.appregistry`` label.

After a successful build, if :ref:`OMPS integration <omps-integration>`
is enabled, operator manifests are uploaded into configured application
registry and namespace.

Details on how operator manifest can be accessed from the application registry are
stored in koji build, in section ``build.extra.operator_manifests.appregistry``.

Manifests will not be pushed to the application registry for scratch builds,
isolated builds, or re-builds to prevent unwanted changes


Building Source Container Images
================================

OSBS is able to build source container image from a particular koji build
previously created by OSBS. To create a source container build you have to
specify either koji N-V-R or build ID for the image build you want to
create a source container image for.

When koji build is using lookaside cache, that may include all sort of things about which
we can't get any information, in that case source container build will fail.

Under the hood the BSI_ project is used to generate source images from
sources identified and collected by OSBS. Please note that BSI script must be
available in the OSBS buildroot as `bsi` executable in `$PATH`.


Current limitations:

* only Source RPMs and sources fetched through :ref:`cachito-integration` are added into source container image
* only koji internal RPMs are supported

Support for other types of sources and external builds will be added in future.

.. _`BSI`: https://github.com/containers/BuildSourceImage


Signing intent resolution
-------------------------

Resolution of signing intent is done in following order:

* signing intent specified from CLI params (koji, osbs-client),
* otherwise signing intent is taken from the original image build,
* if undefined then default signing intent from `odcs` configuration section in `reactor-config-map` is used.

If ODCS integration is disabled, unsigned packages are allowed by default.


Koji integration
----------------

Koji integration must be enabled for building source container images.
Source container build requires metadata stored in koji builds and koji
database of RPM builds which source container build uses to lookup for sources.

Source container builds uses different task type: `buildSourceContainer`.


Koji Build Metadata Integration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Source container build uses metadata from specified image build in the following manner:

* **name**: suffix `-source` is appended to original name (`ubi8-container` will be transformed to `ubi8-container-source`)
* **version**: value is the same as original image build
* **release**: a suffix `.X` is appended to original release value, where `X` is a sequential integer starting from 1 increased by OSBS for each source image rebuild.

For example, from N-V-R `ubi8-container-8.1-20` OSBS creates source container build `ubi8-container-source-8.1-20.1`.

The original image N-V-R is stored in `extra.image.sources_for_nvr` attribute in koji source container build metadata.


Building source container images using koji
-------------------------------------------

Using a koji client CLI directly you have to specify git repo URL and branch::

    koji source-container-build <target>  --koji-build-nvr=NVR --koji-build-id=ID

For a full list of options::

    koji source-container-build --help


Building source container images using osbs-client
--------------------------------------------------

Please note that mainly ``koji`` and ``fedpkg`` commands should be used
for building container images instead of direct ``osbs-client`` calls.

To execute build via osbs-client CLI use::

    osbs build-source-container -c <component> -u <username> --sources-for-koji-build-nvr=N-V-R --sources-for-koji-build-id=ID

To see full list of options execute::

    osbs build-source-container --help

To see all osbs-client subcommands execute::

    osbs --help

Please note that ``osbs-client`` must be configured properly using config file ``/etc/osbs.conf``.
Please refer to :ref:`osbs-client configuration section <configuring-osbs-client>`
for configuration examples.
