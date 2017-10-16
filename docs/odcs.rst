ODCS Integration
================

When a container build is started, the user usually provides the URL to a yum
repo that contains the packages to be installed when building the container
image. Usually, this is a puddle containing signed packages.

In the case a yum repo URL is not provided, and Koji integration is enabled,
OSBS falls back to the yum repo associated with the build tag for the Koji build
target (required parameter for Koji container builds). Packages in this yum repo
are not always signed. This is not ideal as it’s trivial to produce a container
image that should never be shipped.

Another problem with the current state is that to produce an image with signed
packages, a parameter is needed (yum repo URL). To complicate matters, this yum
repo URL will likely be different for future builds. This poses a huge problem
for OSBS autorebuilds. In this case, OSBS will automatically rebuild layered
images upon detected changes to parent image. It is necessary that these
rebuilds pick up the latest, most relevant, signed packages.

The On Demand Compose Service (`odcs`_) project has been created to address
current limitations in obtaining a yum repo with signed packages. The general
idea is to not require the user to provide a yum repo url when building
container images.  Instead, OSBS will use ODCS to generate a yum repo with the
relevant signed packages based on user configuration.

API Changes
-----------

The following parameters will be used by ``buildContainer`` task defined by
``koji-containerbuild`` plugin:

``odcs_signed <bool, default True>`` When set to False, OSBS will provide an
empty list of ``sigkeys`` to ODCS. An empty list of ``sigkeys`` indicates the
signature of packages will not be checked causing the compose to potentially
include unsigned packages. If ODCS integration is not in place, this parameter
has no effect.

``container.yaml`` Changes
--------------------------

In git repo, alongside Dockerfile, the ``container.yaml`` file will be enhanced
to contain a "compose" section. This section will include all the required
information for requesting a compose to be created by ODCS::

    compose:
      # Optional, default False. Inherit compose used for building parent image
      inherit: true
      # Optional, default "tag".
      type: tag
      # Required for "tag" type. Must contain at least one item.
      packages:
      - package-1-name # This is a package name, not an NVR.
      - package-2-name
      # Optional. Default value is configured in OSBS environment
      sigkeys:
      - AB123456

Alternatively, for modules support::

    compose:
      type: module
      # Eventually, ODCS will support ":" separator, for now "-" is used:
      #     https://pagure.io/odcs/issue/98
      # Required for "module" type. Must contain at least one item.
      modules:
      - "module_name1:stream1"
      - "module_name2:stream1"

Behavior Changes
----------------

During an OSBS build, if ``container.yaml`` exists **and** ``compose`` key is defined:

- OSBS orchestrator build will request a compose from ODCS based on specified
  parameters in ``compose`` section. Once compose completes, build resumes and
  the URL for the generated compose is passed to worker builds via
  ``yum_repourls`` build parameter. *NOTE: If odcs-signed parameter was set to
  false, the compose request will ignore the sigkeys provided in
  container.yaml.*

- If ``compose`` section sets ``inherit`` to ``true``, the ODCS composes used in
  building the parent image will be re-used. This may require OSBS to request
  compose to be regenerated if it’s in *remove* state. Once compose is
  available, build resumes and the URL for the generated compose is passed to
  worker builds via ``yum_repourls`` build parameter. *NOTE: this can be used in
  conjunction with the method stated on previous bullet point. In which case,
  multiple ODCS composes are used.*

- In the Koji build metadata, ``build.extra.image.odcs.compose_ids`` will be a
  list of each ODCS compose used. Additionally,
  ``build.extra.image.odcs.signed`` will correspond to the value of
  ``odcs_signed`` build parameter which defaults to ``True``

Otherwise, ODCS integration is completely bypassed. The previous behavior of
using Koji yum repo for build tag of build target is used by default. If
``yum_repourls`` parameter is used, ODCS integration is also bypassed and only
the given yum-repos are used. In either case, the Koji build metadata will not
include any of the ``build.extra.image.odcs.*`` keys.

To clarify, by the time the OSBS worker builds are created, if ODCS is to be
used, a yum repo URL will be available. This will then be passed to worker
builds via the usual yum_repourls parameter, ensuring all worker builds use the
same exact compose.

Relevant Projects
-----------------

- `koji-containerbuild`_

- `atomic-reactor`_

- `osbs-client`_

- `odcs`_

- `koji`_

.. _`koji-containerbuild`: https://github.com/release-engineering/koji-containerbuild
.. _`atomic-reactor`: https://github.com/projectatomic/atomic-reactor
.. _`osbs-client`: https://github.com/projectatomic/osbs-client
.. _`odcs`: https://pagure.io/odcs
.. _`koji`: https://pagure.io/koji
