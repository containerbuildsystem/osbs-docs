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

Concepts
--------

Signing Intent
""""""""""""""

When an ODCS compose is used, the idea of a signing intent is used. This refers
to the signature of the content to be included in the container image. For
instance, an environment may provide the following intents: ``release``,
``beta``, and ``unsigned``. Each one of those intents is then mapped to a list
of signing keys. These signing keys are then used during compose creation. The
packages to be included must have been signed by any of the keys listed. In the
example above, the intents could be mapped to the following keys::

    # Only include packages that have been signed by "my-release-key"
    release -> my-release-key
    # Include packages that have been signed by either "my-beta-key" or
    # "my-release-key"
    beta -> my-beta-key, my-release-key
    # Do not check signature of packages - may include unsigned packages
    unsigned -> <empty>

This mapping will be defined in the secret named by reactor_config_secret which
will allow updates to be reflected instantly for automatically triggered builds
from an existing BuildConfig. For example::

    signing_intents:
    - release:
      - 'my-release-key'
    - beta:
      - 'my-beta-key'
      - 'my-release-key'
    - unsigned: []

The definition of intents also describes the restrictive order, which will be
enforced when building layered images. For instance, consider the case of two
images, X and Y. Y uses X as its parent image (FROM X). If image X was built
with "beta" intent, image Y's intent can only be "beta" or "unsigned". If the
dist-git repo for image Y has it configured to use "release" intent, this value
will be downgraded to "beta" at build time.

Automatically downgrading the signing intent, instead of failing the build, is
important for allowing a hierarchy of layered images to be built automatically
by ImageChangeTriggers. For instance, with Continuous Integration in mind, a
user may want to perform daily builds without necessarily requiring signed
packages while periodically also producing builds with signed content. In this
case, the ``signing_intent`` in ``container.yaml`` can be set to ``release`` for
all the images in hierarchy.  Whether or not the layered images in the hierarchy
use signed packages can be controlled by simply overriding the signing intent of
the top most ancestor image. The signing intent of the layered images would then
be automatically adjusted as needed.

In the case where multiple composes are used, the least restrictive intent is
used. Continuing with our previous signing intent example, let's say a container
image build request uses two composes. Compose 1 was generated with no signing
keys provided, and compose 2 was generated with "my-release-key". In this case,
the intent is "unsigned".

Because compose IDs can be passed in to OSBS, a mechanism will be in place to
classify the intent of an existing compose. This will be done by inspecting the
signing keys used for generating the compose and performing a reverse mapping to
determine the intent. If a match cannot be determined, intent will be set to
``None``, which is less restrictive than any defined signing intents.

The ``signing_intent`` specified in ``container.yaml`` can be overridden with
the build parameter of same name. This particular parameter will be ignored for
autorebuilds. The value in ``container.yaml`` should always be used in that
case. Note that the signing intent used by the compose of parent image is still
enforced.

The Koji build metadata will contain a new key,
``build.extra.image.odcs.signing_intent_overridden``, to indicate whether or not
the ``signing_intent`` was overridden (CLI parameter, automatically downgraded,
etc).  This value will only be ``true`` if
``build.extra.image.odcs.signing_intent`` does not match the ``signing_intent``
in ``container.yaml``.

Enabling ODCS Integration
"""""""""""""""""""""""""

The first step in enabling ODCS integration is by configuring the OSBS
environment to use a specific ODCS server via the ``odcs_*`` configuration
parameters.

Next, ``container.yaml`` must be created and it must include the *compose*
section. See example further down. This file must then be committed and pushed
to git repository. Simply building the container image as usual will trigger
ODCS interaction.


ODCS Compose Checks
"""""""""""""""""""

When an ODCS compose ID is passed in as an argument, OSBS will perform some
checks to ensure the compose is usable.

- State done: verify the compose is not about to expire, extending if needed.
  Continue with OSBS build once compose is in this state.

- State failed: not supported, causes failure of the OSBS build.

- State removed: request compose to be regenerated. In the future, we may want
  to check the ``pre-remove`` state: https://pagure.io/odcs/issue/90

- State wait|generating: wait for a certain amount of time (configurable in OSBS
  environment) for compose request to move out of this state. Polling is
  performed during wait period.

API Changes
-----------

- ``signing_intent <str, default None>``

  Exposed in `koji-containerbuild`_, and `osbs-client`_.

  Must not be used if ODCS integration is not enabled.

  Overwrites ``signing_intent`` in ``container.yaml``.

- ``compose_ids <list of ints>``

  Exposed in `koji-containerbuild`_, and `osbs-client`_.

  May not be used with ``yum_repourls`` parameter.

  May not be used if ODCS integration is not enabled.

  May not be used with ``signing_intent`` parameter.

  Ignores *compose* section in ``container.yaml`` and does not request a new
  ODCS compose to be created. The provided composes are used instead.


CLI Changes
-----------

These mostly correspond to the API changes above. It's listed here mainly to
emphasize they may be spelled differently.

- ``--signing-intent=<str, default None>``

  Exposed in `koji-containerbuild`_, `rpkg`_, and `osbs-client`_.

  Same API restrictions apply.


- ``--compose-id=<int, default None, may be used multiple times>``

  Exposed in `koji-containerbuild`_, `rpkg`_, and `osbs-client`_.

  Same API restrictions apply.


``container.yaml`` Changes
--------------------------

In git repo, alongside Dockerfile, the ``container.yaml`` file will be enhanced
to contain a *compose* section. This section will include all the required
information for requesting a compose to be created by ODCS::

    compose:
      # Required for ODCS Koji "tag" type usage. Must contain at least one item.
      packages:
      - package-1-name # This is a package name, not an NVR.
      - package-2-name
      # Optional. Default and possible values are configured in OSBS
      # environment.
      signing_intent: release

Alternatively, for modules support::

    compose:
      # Eventually, ODCS will support ":" separator, for now "-" is used:
      #     https://pagure.io/odcs/issue/98
      # Required for ODCS "module" type usage. Must contain at least one item.
      modules:
      - "module_name1:stream1"
      - "module_name2:stream1"

**Exactly one non-empty list of "modules" or "packages" must be provided. If
both or none are defined build will fail.**

Behavior Changes
----------------

During an OSBS build, if ``container.yaml`` exists **and** *compose* key is
defined:

- OSBS orchestrator build will request a compose from ODCS based on specified
  parameters in *compose* section. Once compose completes, build resumes and
  the compose ID is passed to the worker builds via the new ``compose_ids``
  build parameter.

- In the Koji build metadata:

    - ``build.extra.image.odcs.compose_ids``: list of each ODCS compose used.

    - ``build.extra.image.odcs.signing_intent``: final signing intent of the
      ODCS composes after adjusting for CLI parameter, automatically downgraded,
      etc.

    - ``build.extra.image.odcs.signing_intent_overridden``: whether or not the
      signing intent used is different than the one defined in
      ``container.yaml``.

Otherwise, ODCS integration is completely bypassed. The previous behavior of
using Koji yum repo for build tag of build target is used by default. If
``yum_repourls`` parameter is used, ODCS integration is also bypassed and only
the given yum repos are used. In either case, the Koji build metadata will not
include any of the ``build.extra.image.odcs.*`` keys.

To clarify, by the time the OSBS worker builds are created, if ODCS is to be
used, the composes have been created and are ready to be used. The IDs for the
composes will then be passed to worker builds via the new ``compose_ids``
parameter, ensuring all worker builds use the same exact composes.

Relevant Projects
-----------------

- `koji-containerbuild`_

- `atomic-reactor`_

- `osbs-client`_

- `odcs`_

- `koji`_

.. _`atomic-reactor`: https://github.com/projectatomic/atomic-reactor
.. _`koji-containerbuild`: https://github.com/release-engineering/koji-containerbuild
.. _`koji`: https://pagure.io/koji
.. _`odcs`: https://pagure.io/odcs
.. _`osbs-client`: https://github.com/projectatomic/osbs-client
.. _`rpkg`: https://pagure.io/rpkg
