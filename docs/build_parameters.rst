.. _build_parameters:

Build Parameters
================

Environment vs User Parameters
""""""""""""""""""""""""""""""

atomic-reactor requires various parameters to build container images. These
range from the Koji Hub URL to git commit. We can categorize them into
environment and user parameters. The main difference between them is how they
are reused. Environment parameters are shared by different build requests, while
user parameters are unique to each build request.

Environment parameters are used for configuring usage of external services such
as Koji, Pulp, ODCS, SMTP, etc. They are also used for controlling some aspects
of the container images built, for example, distribution scope, vendor,
authoritative-registry, etc. These may change over time as the environment
changes.

User parameters contain the unique information for a user's build request: git
repository, git branch, Koji target, etc. These should be reused for
autorebuilds and are not affected by environment changes.


Reactor Configuration
"""""""""""""""""""""

As of Arrangement 6, environment configuration is provided to build containers
in a way that is less coupled to the ``Build``/``BuildConfig`` objects. The
pre-build plugin ``reactor_config`` determines and provides all
environment configuration to other plugins.

Previously, the value of ``reactor_config`` was mounted into container as a
secret. With Arrangement 6 it is supplied as a ``ConfigMap`` like::

    apiVersion: v1
    kind: ConfigMap
    data:
        "config.yaml": <encoded yaml>

For an orchestrator build, the ``ConfigMap`` is mapped via the downward API
into the **REACTOR_CONFIG** environment variable in a build container.
When the ``BuildConfig`` is instantiated, OpenShift selects the ``ConfigMap``
with name given in  **reactor-config-map**, retrieves the contents of the key
**config.yaml**, and sets it as the value of the **REACTOR_CONFIG** environment
variable in the build container.

For worker builds, the **REACTOR_CONFIG** environment variable is defined
as an inline **value** (via the
**reactor_config_override** build parameter in osbs-client). To populate this
parameter, the ``orchestrate_build`` plugin uses the ``reactor_config``
plugin to read the reactor configuration for the orchestrator build, using it as
the basis of the reactor configuration for worker builds with the following
modifications:

- **openshift** section is replaced with worker specific values. These
  values can be read from the osbs-client ``Configuration`` object created for
  each worker cluster.
- **worker_token_secrets** is completely removed. This section is intended
  for orchestrator builds only.

The schema definition `config.json`_ in atomic-reactor contains a description
for each property.

Example of **REACTOR_CONFIG**::

    version: 1

    clusters:
        x86_64:
        - name: x86_64-worker-1
          max_concurrent_builds: 15
          enabled: True
        - name: x86_64-worker-2
          max_concurrent_builds: 6
          enabled: True

    clusters_client_config_dir: /var/run/secrets/atomic-reactor/client-config-secret

    koji:
        hub_url: https://koji.example.com/hub
        root_url: https://koji.example.com/root
        auth:
            ssl_certs_dir: /var/run/secrets/atomic-reactor/kojisecret

    pulp:
        name: my-pulp
        auth:
            ssl_certs_dir: /var/run/secrets/atomic-reactor/pulpsecret

    odcs:
        api_url: https://odcs.example.com/api/1
        auth:
            ssl_certs_dir: /var/run/secrets/atomic-reactor/odcssecret
        signing_intents:
        - keys: ['R123', 'R234']
          name: release
        - keys: ['B123', 'B234', 'R123', 'R234']
          name: beta
        - keys: []
          name: unsigned
        default_signing_intent: release

    smtp:
        host: smtp.example.com
        from_address: osbs@example.com
        error_addresses:
        - support@example.com
        domain: example.com
        send_to_submitter: True
        send_to_pkg_owner: True

    pdc:
        api_url: https://pdc.example.com/rest_api/v1

    arrangement_version: 6

    artifacts_allowed_domains:
    - download.example.com/released
    - download.example.com/candidates

    image_labels:
        vendor: "Spam, Inc."
        authoritative-source-url: registry.public.example.com
        distribution-scope: public

    image_equal_labels:
    - [description, io.k8s.description]

    openshift:
        url: https://openshift.example.com
        auth:
            enable: True
        build_json_dir: /usr/share/osbs/

    group_manifests: False

    platform_descriptors:
    - platform: x86_64
      architecture: amd64
      enable_v1: True

    content_versions:
    - v1
    - v2

    registries:
    - url: https://container-registry.example.com/v2
      auth:
        cfg_path: /var/run/secrets/atomic-reactor/v2-registry-dockercfg

    source_registry:
        url: https://registry.private.example.com

    sources_command: "fedpkg sources"

    required_secrets:
    - kojisecret
    - pulpsecret
    - odcssecret
    - v2-registry-dockercfg
    - client-config-secret

    worker_token_secrets:
    - x86-64-worker-1
    - x86-64-worker-2

    default_image_build_method: imagebuilder


Atomic Reactor Plugins and Arrangement Version 6
""""""""""""""""""""""""""""""""""""""""""""""""

Prior to Arrangement 6, atomic-reactor plugins received environment parameters
as their own plugin parameters. Arrangement 6 was introduced to indicate that
plugins should retrieve environment parameters from **reactor_config** instead.
Plugin parameters that are really environment parameters have been
made optional.

The osbs-client configuration **reactor_config_map** defines
the name of the ``ConfigMap`` object holding **reactor_config**. This
configuration option is mandatory for arrangement versions greater than or
equal to 6. Previous osbs-client configuration **reactor_config_secret**
is deprecated.

An osbs-client build parameter **reactor_config_override**
allows reactor configuration to be passed in as a python dict. It is
also validated against `config.json`_ schema. When both
**reactor_config_map** and **reactor_config_override** are defined,
**reactor_config_override** takes precedence. NOTE: **reactor_config_override**
is a python dict, not a string of serialized data.

Creating Builds
"""""""""""""""

osbs-client no longer renders the atomic-reactor plugin configuration
at ``Build`` creation.
Instead, the **USER_PARAMS** environment variable is set on the ``Build``
containing only user parameters as JSON. For example::


    {
        "build_type": "orchestrator",
        "git_branch": "my-git-branch",
        "git_ref": "abc12",
        "git_uri": "git://git.example.com/spam.git",
        "is_auto": False,
        "isolated": False,
        "koji_task_id": "123456",
        "platforms": ["x86_64"],
        "scratch": False,
        "target": "my-koji-target",
        "user": "lcarva",
        "yum_repourls": ["http://yum.example.com/spam.repo", "http://yum.example.com/bacon.repo"],
    }


Rendering Plugins
"""""""""""""""""

Once the build is started, control is handed over to atomic-reactor. Its input
plugin ``osv3`` looks for the environment variable **USER_PARAMS** and uses the
osbs-client method ``render_plugins_configuration`` to generate the plugin
configuration on the fly.  The generated plugin configuration contains the
order in which plugins will run as well as user parameters.


Secrets
"""""""

Because the plugin configuration renders at build time (after ``Build``
object is created), we cannot select which secrets to mount in container
build based on which plugins have been enabled. Instead, all the secrets that
may be needed must be mounted. The **reactor_config** ``ConfigMap`` defines
the full set of secrets it needs via its **required_secrets** list.

When orchestrator build starts worker builds, it uses the same set of secrets.
This requires worker clusters to have the same set of secrets available. For
example, if **reactor_config** defines::

    required_secrets:
    - kojisecret
    - pulpsecret

Secrets named **kojisecret** and **pulpsecret** must be available in orchestrator and
worker clusters. They don't need to have the same value, just the same name. For
instance, worker and orchestrator builds may use different authentication
certificates.

Secrets needed for communication from orchestrator build to worker clusters are
defined separately in **worker_token_secrets**. These are not passed along
to worker builds.

Site Customization
""""""""""""""""""

The site customization configuration file is no longer read from the system
creating the OpenShift ``Build`` (usually koji builder). Instead, this
customization file must be stored and read from inside the builder image.


.. _`config.json`: https://github.com/projectatomic/atomic-reactor/blob/master/atomic_reactor/schemas/config.json

