Isolated Builds
===============

When a build is created via OSBS, the built container image is pushed to the
container registry updating various tag references in the container registry.
Some of these tag references are unique, while others are meant to transition
over time to reference a newer build, for instance ``latest`` and ``{version}``
tags. In other words, building a container image will always update these
transient, non-unique, tags in container registry. In some cases, this is not
desired.

Consider the case of a container image that includes one, or more, packages that
have recently been identified to contain a security vulnerability. To address
this issue, a new container image must be built. The difference for this build
is that only changes related to the security fix must be applied. Any unrelated
development that has occurred should be ignored. It would not be correct to
update the ``latest`` tag reference with this build. To achieve this, the concept
of isolated builds is introduced.

API Changes
-----------

The following parameters will be used by ``buildContainer`` task defined by
``koji-containerbuild`` plugin:

``koji_parent_build <nvr_str or build_id_str>`` When defined, OSBS will
determine the container image tag reference used for the given Koji build, and
set it as the value for the ``FROM`` instruction in Dockerfile of container
image to be built.  The ``name`` portion of the tag reference for the given
parent Koji build must match the ``name`` portion of the existing (to be
overwritten) tag reference in the Dockerfile.

``release <release_str>`` When defined, OSBS will use value to overwrite
whichever release value may be defined in Dockerfile . This value will be
visible in the Koji build created, as well as, the ``{version}-{release}`` tag
reference.

``isolated <bool>`` When defined, OSBS will only tag image with unique and
``{version}-{release}`` tags. It will also require a release value to be given
via ``release`` parameter which must match the format: ``^\d+\.\d+(\..+)?$``

Builds can be initiated via ``buildContainer`` API method in Koji, provided by
koji-containerbuild plugin::

    source = "{git-url}#{git-reference}"
    target = "{koji-target}"
    opts = {
        "git_branch": "{git-branch}",
        "yum_repourls": ["{url-for-yum-repo}", ...],
        "isolated": True,
        "koji_parent_build": "{parent-build-nvr}",
        "release": "{release}",
    }

    koji_xmlrpc.buildContainer(source, target, opts)


Implementation Details
----------------------

Plugins
'''''''

inject_parent_image
"""""""""""""""""""

This new pre build plugin will take a ``koji_parent_build`` parameter. If the
parameter has a truthy value, it queries Koji for information about the build
and determine the unique tag reference for such build. This information is
retrieved by inspecting the archives attached to Koji build. The first archive
containing the key path ``'extra'.'docker'.'repositories'`` will be used. The
repository reference to be used is the unique manifest digest (contains sha256).

This plugin runs in both orchestrator and worker builds just before the
``pull_base_image`` plugin.

This plugin is disabled if a custom base image is being built.

bump_release
""""""""""""

The ``bump_release`` plugin is modified to strip off any numerical ``.N``
suffixes from the return value of Koji’s ``getNextRelease`` API call. When an
isolated build is created, a release number will be given in the format:
``20.1``. This indicates that this is the first patch to the container image
that has release ``20``. The next time a non-isolated build is requested,
``getNextRelease`` will return ``21.1``.  Over time, this number might be quite
high as the patched container images pile up. To handle this, ``bump_release``
adjusts what ``getNextRelease`` returns::

    20       -> 20
    20.1     -> 21  # Adjusted
    20.f25   -> 20.f25
    20.1.f25 -> 21.f25  # Adjusted

It then verifies that this adjusted value indeed generates a unique NVR. If not,
value is incremented until a unique NVR is generated (this is also the approach
taken for handling failed builds).

orchestrate_build
"""""""""""""""""

No changes to this plugin are required, but when rendered it must be given the
``koji_parent_build`` and ``isolated`` parameters so they can be passed to the
worker builds.

OSBS Client API
'''''''''''''''

The osbs-client API method ``create_orchestrator_build`` is enhanced to take the
additional build parameters: ``koji_parent_build (str)``, ``isolated (bool)``.
The API method ``create_worker_build`` needs support only for the
``koji_parent_build (str)`` parameter. They should pass along the values to the
build spec so it can be used for rendering the build request.

The ``release`` value should already be accepted by both
``create_orchestrator_build`` and ``create_worker_build``.

If ``isolated`` parameter is given, ``tag_from_config`` in orchestrator build
must be configured to not set any transient tags: ``{version}`` and ``latest``.
It should only contain the unique tag, and the ``{version}-{release}`` primary
tag.

If both ``isolated`` and ``scratch`` parameters are given, an exception should
be raised to prevent this unsupported scenario.

If ``isolated`` parameter is given but ``release`` parameter is not, an
exception should be raised to prevent this unsupported scenario.

If ``isolated`` and ``release`` parameters are given, the release value must
match the format: ``^\d+\.\d+(\..+)?$`` For instance, 20.1, 20.2, 20.1.f25

A new ``_create_isolated_build`` method is called whenever ``isolated``
parameter is ``True``. This creates a build directly, without a ``BuildConfig``.
The name of the build takes into account the expected NVR to avoid conflicting
builds.

To support build prioritization a node selector for isolated builds is also
be added. This can be done after initial release of this feature.

Koji Container Build
''''''''''''''''''''

The plugin koji-containerbuild takes the new ``koji_parent_build``,
``isolated``, and ``release`` parameters. The parameters are then forwarded to
``create_orchestrator_build`` osbs-client API method.

If ``release`` parameter is given, koji-containerbuild verifies release value is
unique raising exception if not.

Further parameter validation is performed by osbs-client library once
``create_orchestrator_build`` is invoked.

Relevant Projects
-----------------

- `koji-containerbuild`_

- `atomic-reactor`_

- `osbs-client`_

- `rpkg`_

- `fedpkg`_

- `freshmaker`_

- `koji`_

.. _`koji-containerbuild`: https://github.com/release-engineering/koji-containerbuild
.. _`atomic-reactor`: https://github.com/projectatomic/atomic-reactor
.. _`osbs-client`: https://github.com/projectatomic/osbs-client
.. _`rpkg`: https://pagure.io/rpkg
.. _`fedpkg`: https://pagure.io/fedpkg
.. _`freshmaker`: https://pagure.io/freshmaker
.. _`koji`: https://pagure.io/koji/
