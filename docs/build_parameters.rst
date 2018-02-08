Build Parameters
================

OSBS’s autorebuild feature is meant to automatically start new builds of layered
images whenever the parent image changes. This is particularly useful for image
owners that maintain a large hierarchy of images. Currently, they need to
manually start each image build directly and in the correct order.

The goal of autorebuilds is to remove the need for image owners to start each
required image build. Instead, they can start a build for the topmost ancestor
which upon completion triggers the next level of layered images, and so on.

In the current design of osbs-client and atomic-reactor, the parameters needed
for rebuilding the container image are stored in the corresponding
``BuildConfig`` object in OpenShift. It contains required information such as
build image, atomic reactor plugins configuration, secrets to be mounted,
``ImageChange`` triggers, git information (repository, commit, branch), and Koji
task ID.

A code release of OSBS tooling means changing the build image to be used for
building containers.  Any new container image build will use the newer build
image as expected. In the case of autorebuilds, the ``BuildConfig`` objects are
not updated. When OpenShift detects a change in parent image and starts a new
build for the layered image, it will use the outdated build image. To work
around this limitation, image owner must start explicit builds for each image in
the hierarchy of images to update the underlying ``BuildConfig`` object to use
the newer build image. This contradicts our previously stated goal. This
document outlines changes required to approach that goal.


Design
------

There are multiple changes needed to achieve the goal of non-disruptive code
releases. Some parts can be done by changing the process while others require
large design changes to osbs-client and atomic-reactor.

Latest Released Build Image
"""""""""""""""""""""""""""

The build image used for building container images is defined in the
``Build``/``BuildConfig`` OpenShift objects under
``.spec.strategy.customStrategy.from`` object. This can be a full reference to a
specific container image in a container registry; or it can reference an
ImageStreamTag object.

To avoid having to update each ``BuildConfig`` object to use a newer build
image, use transient tags for the build image. Transient tags are those that
are meant to reference different images over time. To use a newer build image,
simply move the transient tag to reference it. For instance, say we have an
``ImageStream`` called **my-build-image** which tracks a remote container
repository. The transient tag **released** is used to track the latest build
image that should be used. The ``BuildConfig`` objects define the
``ImageStreamTag`` object **my-build-image:released** is used to track the build
image. To start using a newer build image, simply tag the newer build image with
**released** tag.

This can be done when either ``DockerImage``, or ``ImageStreamTag`` types are
used.

Environment vs User Parameters
""""""""""""""""""""""""""""""

atomic-reactor requires various parameters to build container images. These
range from the Koji Hub URL to git commit. We will categorize them in two,
environment and user parameters. The main difference between them is how they
are reused. Environment parameters are shared by different build requests, while
user parameters are unique to each build request.

Environment parameters are used for configuring usage of external services such
as Koji, Pulp, ODCS, SMTP, etc. They are also used for controlling some aspects
of the container images built, for example, distribution scope, vendor,
authoritative-registry, etc. These may change over time as the environment
changes.

User parameters contain the unique information for a user's build request, git
repository, git branch, Koji target, etc. These should be reused for
autorebuilds and are not affected by environment changes.


Reactor Configuration
"""""""""""""""""""""

Environment configuration should be mounted in the build containers. This
decouples the configuration from the ``Build``/``BuildConfig`` objects. The
existing pre-build plugin ``reactor_config`` will be enhanced to provide all
environment configuration to other plugins.

Currently, the value of ``reactor_config`` is mounted into container as a
secret. At the time it was implemented it wasn't clear how to make a
``ConfigMap`` available in a build container. It is possible to do so by using
an environment variable with ``valueFrom``::

    apiVersion: v1
    kind: BuildConfig
    spec:
      strategy:
        customStrategy:
          env:
          - name: REACTOR_CONFIG
            valueFrom:
              configMapKeyRef:
                name: reactor-config-map
                key: config.yaml

When the ``BuildConfig`` is instantiated, OpenShift will select the ``ConfigMap``
of name **reactor-config-map**, retrieve the contents of the key
**config.yaml**, and set it as the value of the **REACTOR_CONFIG** environment
variable in the build container. For instance::

    apiVersion: v1
    kind: ConfigMap
    data:
        "reactor-config": <base64 encoded yaml>

For worker builds, the **REACTOR_CONFIG** environment variable will be defined
inline via **value**, instead of **valueFrom**. The ``orchestrate_build`` plugin
will read the environment variable from its own running environment, encode its
value in base64, and set environment variable **REACTOR_CONFIG** in the
``Build``/``BuildConfig`` object for each worker build. This will ensure
environment parameters are used consistently throughout orchestrator and worker
builds. The plugin will also substitute any environment configuration if needed.
As an example, the **openshift** section will be replaced with the worker
OpenShift cluster information.

The base64 encoded YAML will match the schema described in
``atomic_reactor/schemas/config.json``. The following properties will be added
to the root of the object::

    "koji": {
        "description": "Koji instance",
        "type": "object",
        "properties": {
            "hub_url": {
                "description": "Koji hub's xmlrpc url",
                "type": "string"
            },
            "root_url": {
                "description": "Koji's root storage url",
                "type": "string"
            },
            "auth": {
                "description": "Authentication information",
                "type": "object",
                "properties": {
                    "proxyuser": {
                        "description": "Login as different user",
                        "type": "string"
                    },
                    "ssl_certs_dir": {
                        "description": "Path to directory with cert and ca files",
                        "type": "string"
                    },
                    "krb_cache_path": {
                        "description": "Path to kerberos credential cache file",
                        "type": "string"
                    },
                    "krb_principal": {
                        "description": "Kerberos principal",
                        "type": "string"
                    },
                    "krb_keytab_path": {
                        "description": "Location of Kerberos keytab, e.g. FILE:<absolute_path>",
                        "type": "string"
                    }
                },
                "additionalProperties": false,
                "anyOf": [
                    {
                        "required": ["ssl_certs_dir"]
                    },
                    {
                        "required": ["krb_principal", "krb_keytab_path"]
                    },
                    {
                        "required": []
                    }
                ]
            },
            "additionalProperties": false,
            "required": ["hub_url", "root_url", "auth"]
        }
    },
    "pulp": {
        "description": "Pulp registry instance",
        "type": "object",
        "properties": {
            "name": {
                "description": "Pulp registry name, specified in /etc/dockpulp.conf",
                "type": "string"
            },
            "auth": {
                "description": "Authentication information",
                "type": "object",
                "properties": {
                    "ssl_certs_dir": {
                        "description": "Path to directory with pulp.cer and pulp.key files",
                        "type": "string"
                    },
                    "username": {
                        "description": "Pulp username",
                        "type": "string"
                    },
                    "password": {
                        "description": "Pulp password",
                        "type": "string"
                    }
                },
                "additionalProperties": false,
                "anyOf": [
                    {
                        "required": ["ssl_certs_dir"]
                    },
                    {
                        "required": ["username", "password"]
                    },
                ]
            }
        },
        "additionalProperties": false,
        "required": ["name", "auth"]
    },
    "odcs": {
        "description": "OnDemand Compose Service (ODCS) instance",
        "type": "object",
        "properties": {
            "api_url": {
                "description": "ODCS api url, including api version",
                "type": "string"
            },
            "insecure": {
                "description": "Don't check SSL certificate for api_url",
                "type": "boolean"
            },
            "auth": {
                "description": "Authentication information",
                "type": "object",
                "properties": {
                    "ssl_certs_dir": {
                        "description": "Path to directory with cert file",
                        "type": "string"
                    },
                    "openidc_dir": {
                        "description": "Path to directory with token file",
                        "type": "string"
                    }
                },
                "additionalProperties": false,
                "anyOf": [
                    {
                        "required": ["ssl_certs_dir"]
                    },
                    {
                        "required": ["openidc_dir"]
                    }
                ]
            }
        },
        "additionalProperties": false,
        "required": ["api_url", "auth"]
    },
    "smtp": {
        "description": "SMTP notifications",
        "type": "object",
        "properties": {
            "host": {
                "description": "SMTP host server name",
                "type": "string"
            },
            "from_address": {
                "description": "From email address in notification email",
                "type": "string"
            },
            "additional_addresses": {
                "description": "Email addresses to always send notification email",
                "type": "array",
                "items": {
                    "type": "string"
                }
            },
            "error_addresses": {
                "description": "Email addresses to send notifications if no other email address can be resolved",
                "type": "array",
                "items": {
                    "type": "string"
                }
            },
            "domain": {
                "description": "Domain used when constructing email addresses",
                "type": "string"
            },
            "send_to_submitter": {
                "description": "Send email notification to Koji task submitter",
                "type": "boolean"
            },
            "send_to_pkg_owner": {
                "description": "Send email notification to Koji package owner",
                "type": "boolean"
            },
            "additionalProperties": false,
            "required": ["url", "from_address"]


        }
    },
    "pdc": {
        "description": "Product Definition Center (PDC) instance",
        "type": "object",
        "properties": {
            "api_url": {
                "description": "PDC api url, including api version",
                "type": "string"
            },
            "insecure": {
                "description": "Don't check SSL certificate for api_url",
                "type": "boolean"
            }
        },
        "additionalProperties": false,
        "required": ["api_url"]
    },
    "arrangement_version": {
        "description": "Arrangement version",
        "type": "integer"
    },
    "artifacts_allowed_domains": {
        "description": "Domains allowed to use when fetching artifacts by url",
        "type": "array",
        "items": {
            "type": "string"
        }
    },
    "image_labels": {
        "description": "Labels to be applied to container image",
        "type": "object",
        "patternProperties": {
            "^[\w\.-]+$": {"type": "string"}
        },
        "additionalProperties": false
    },
    "image_equal_labels": {
        "description": "Labels that are expected to be equal in value",
        "type": "array",
        "items" : {
            "type": "array",
            "items": {
                "type": "string"
            }
        }
    },
    "openshift": {
        "description": "OpenShift instance running build",
        "type": "object",
        "properties": {
            "url": {
                "description": "OpenShift url",
                "type": "string"
            },
            "insecure": {
                "description": "Don't check SSL certificate for url",
                "type": "boolean"
            }
            "auth": {
                "description": "Authentication information",
                "type": "object",
                "properties": {
                    "enable": {
                        "description": "Enable authentication",
                        "type": "boolean"
                    },
                    "ssl_certs_dir": {
                        "description": "Path to directory with cert, key and ca files",
                        "type": "string"
                    },
                    "krb_cache_path": {
                        "description": "Path to kerberos credential cache file",
                        "type": "string"
                    },
                    "krb_principal": {
                        "description": "Kerberos principal",
                        "type": "string"
                    },
                    "krb_keytab_path": {
                        "description": "Location of Kerberos keytab, e.g. FILE:<absolute_path>",
                        "type": "string"
                    }
                },
                "additionalProperties": false,
                "anyOf": [
                    {
                        "required": ["krb_principal", "krb_keytab_path"]
                    },
                    {
                        "required": []
                    },
                ]
            },
            "additionalProperties": false,
            "required": ["url"]
        },
        "additionalProperties": false,
        "required": ["url"]
    },
    "group_manifests": {
        "description": "Create manifest list, or image index, in container registry"
        "type": "boolean"
    },
    "platform_descriptors": {
        "description": "Definition of supported platforms",
        "type": "array",
        "items": {
            "type": "object",
            "properties": {
                "platform": {
                    "type": "string"
                },
                "architecture": {
                    "type": "string"
                },
                "enable_v1": {
                    "type": "boolean"
                }
            },
            "additionalProperties": false
        }
    },
    "prefer_schema1_digest": {
        "description": "schema 1 as preferred digest",
        "type": "boolean"
    },
    "content_versions": {
        "description": "Produce container image of given version",
        "type": "array",
        "items": {
            "type": "string",
            "enum": ["v1", "v2"]
        }
    },
    "registries": {
        "description": "Container registries to output images",
        "type": "array",
        "items": {
            "type": "object",
            "properties": {
                "url": {
                    "description": "Registry URI including version",
                    "type": "string"
                },
                "insecure": {
                    "description": "Don't check SSL certificate for url",
                    "type": "boolean"
                },
                "auth": {
                    "description": "Authentication information",
                    "type": "object",
                    "properties": {
                        "cfg_path": {
                            "description": "Path to directory  containing .dockercfg for registry auth",
                            "type": "string"
                        }
                    },
                    "additionalProperties": false,
                    "required": ["cfg_path"]
                }
            },
            "additionalProperties": false,
            "required": ["url"]
        }
    },
    "yum_proxy": {
        "description": "Proxy to access yum repositories",
        "type": "string"
    },
    "source_registry": {
        "description": "Container registry to pull parent images",
        "type": "object",
        "properties": {
            "url": {
                "description": "Registry url",
                "type": "string"
            },
            "insecure": {
                "description": "Don't check SSL certificate for url",
                "type": "boolean"
            }
        },
        "additionalProperties": false,
        "required": ["url"]
    },
    "sources_command": {
        "description": "Command to retrieve artifacts in lookaside cache",
        "type": "string"
    },
    "required_secrets": {
        "description": "List of OpenShift secrets required by this configuration",
        "type": "array",
        "items": {
            "type": "string"
        }
    },
    "worker_token_secrets": {
        "description": "List of OpenShift secrets for worker clusters authentication",
        "type": "array",
        "items": {
            "type": "string"
        }
    },
    "build_json_dir": {
        "description": "Path to directory containing Json files templates for osbs-client lib",
        "type": "string"
    }

Example::

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
        vendor: "Spam Inc."
        authoritative-source-url: registry.public.example.com
        distribution-scope: public

    image_equal_labels:
    - [description, io.k8s.description]

    openshift:
        url: https://openshift.example.com
        auth:
            enable: True

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
      auth: /var/run/secrets/atomic-reactor/v2-registry-dockercfg

    source_registry:
        url: https://registry.private.example.com

    sources_command: "fedpkg sources"

    required_secrets:
    - kojisecret
    - pulpsecret
    - odcssecret
    - v2-registry-dockercfg

    worker_token_secrets:
    - x86-64-worker-1
    - x86-64-worker-2

    build_json_dir: /usr/share/osbs/


Secrets
"""""""

Because the plugin configuration will be rendered at build time (after ``Build``
object is created), we no longer can select which secrets to mount in container
build based on which plugins have been enabled. Instead, all the secrets that
may be needed will be mounted. The **reactor_config** ``ConfigMap`` will define
the full set of secrets it needs via its **required_secrets** list.

When orchestrator build starts worker builds, it'll use the same set of secrets.
This requires worker clusters to have the same set of secrets available. For
example, if **reactor_config** defines::

    required_secrets:
    - kojisecret
    - pulpsecret

Secrets named kojisecret and pulpsecret must be available in orchestrator and
worker clusters. They don't need to have the same value, just the same name. For
instance, worker and orchestrator builds may use different authentication
certificates.

Secrets needed for communication from orchestrator build to worker clusters are
defined separately in **worker_token_secrets**. These will not be passed along
to worker builds.

Atomic Reactor Plugins
""""""""""""""""""""""

After **reactor_config** is capable of providing environment parameters, various
atomic reactor plugins will change to retrieve environment parameters from
**reactor_config** instead of taking those values as their own plugin
parameters.

To allow a smoother transition, we'll introduce the new arrangement version 6.
**reactor_config** plugin will provide a helper method so plugins can query the
current arrangement version in use and decide the source of environment
parameters. Plugin parameters that are really environment parameters will be
modified to be optional. Over time, previous arrangement versions will be
removed, and eventually, these plugin parameters can be removed completely, as
well as the arrangement version conditional.


Arrangement Version 6
"""""""""""""""""""""

The purpose of this new arrangement version is to easily identify whether or not
environment parameters are provided by **reactor_config**. The order of plugins
is not expected to change. However, hard coded, or placeholder, environment
parameters in **orchestrator_inner** and **worker_inner** json files will
change.

A new osbs-client configuration **reactor_config_map** will be added to define
the name of the ``ConfigMap`` object holding **reactor_config**. This
configuration option will be mandatory for arrangement versions greater than or
equal to 6. The existing osbs-client configuration **reactor_config_secret**
will be deprecated (for all arrangements).

As any new arrangement version, this will be the default.


Creating Builds
"""""""""""""""

When osbs-client creates a ``Build`` in OpenShift, it also renders the
atomic-reactor plugin configuration which is then stored in ``Build``'s
**ATOMIC_REACTOR_PLUGINS** environment variable. Starting with arrangement
version 6, this will no longer be true. Instead, a new environment variable will
be added to ``Build`` containing only user parameters, **USER_PARAMS**. For
example::


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


Note: **build_type** is currently a symbol (object()). This must be changed to a
string so it can be serialized.

To avoid adding complexity to ``BuildRequest`` class in osbs-client, a new class
will be added, ``BuildRequestV2``. An instance of this class will be returned by
``get_build_request`` API method if **arrangement_version** is greater than or
equal to 6. Otherwise, an instance of the existing ``BuildRequest`` class will
be returned.

``BuildRequestV2`` pseudocode::

    class BuildRequestV2(BuildRequest):

        # Override
        def __init__(...):
            super(..)

            # BuildSpec is not used
            self.spec = None
            self.user_params = BuildUserParams()

        # Override
        def set_params(self, **kwargs):
            # Create BuildUserParams object instead of BuildSpec.

        # Override
        @property
        def inner_template(self):
            raise RuntimeError('inner_template not supported in BuildRequestV2')

        # Override
        @property
        def customize_conf(self):
            raise RuntimeError('customize_conf not supported in BuildRequestV2')

        # Override
        @property
        def dj(self):
            raise RuntimeError('DockJson not supported in BuildRequestV2')

        # The above restrictions will prevent any of the render_* plugin methods
        # from accidentally being called.

        # Override
        def adjust_for_scratch(self):
            # Remove ImageChange triggers
            # Set scratch label
            # Do not handle plugins

        # Override
        def adjust_for_isolated(self):
            # Remove ImageChange triggers
            # Validate release parameter in BuildUserParams
            # Set isolated label
            # Set isolated-release label
            # Do not handle plugins

        # Override
        def adjust_for_custom_base_image(self):
            # Remove ImageChange triggers
            # Do not handle plugins

        # Override
        def render_name(self):
            # Re-implement to use BuildUserParams

        # Override
        def render_node_selectors(self):
            # Re-implement to use BuildUserParams

        # Override
        def render(self):
            # Validate BuildUserParams

            # Render name
            # Render resource limits

            # Set template.spec.source.git.uri
            # Set template.spec.source.git.ref

            # Set template.spec.output.to.name

            # Set template.spec.triggers[0].imageChange.from.name

            # Set template.spec.strategy.customStrategy.from.kind
            # Set template.spec.strategy.customStrategy.from.name

            # Set git-repo-name label
            # Set git-branch label
            # Set koji-task-id label

            # Set template.spec.strategy.customStrategy.env[] USER_PARAMS

            # Adjust for repo info
            # Adjust for scratch build
            # Adjust for isolated build
            # Adjust for custom base image

            # Set required_secrets based on reactor_config
            # Set worker_token_secrets based on reactor_config, if any

            # Log build json
            # Return build json


The new class ``BuildUserParams`` will be added. This class will be similar to
``BuildSpec`` class, but will handle a much smaller set of parameters. It should
also provide a method to convert it to and from json.


Rendering Plugins
"""""""""""""""""

Once the build is started control is handed over to atomic-reactor. Its input
plugin ``osv3`` is responsible for loading the plugin configuration from the
environment variable **ATOMIC_REACTOR_PLUGINS**. If this environment variable is
not found the plugin will look for the environment variable **USER_PARAMS**. If
found, a new code path will generate the plugin configuration on the fly.

The new osbs-client method ``render_plugins_configuration`` will generate the
plugin configuration based on the value of **USER_PARAMS**. As previously
mentioned, environment configuration will be retrieved as needed by each
atomic-reactor plugin. The generated plugin configuration will contain the order
in which plugins will run as well as user parameters.

Support for environment variable **DOCK_PLUGINS** will be removed from ``osv3``.

``render_plugins_configuration`` pseudo code in osbs/api.py::

    class OSBS(object):
        def render_plugins_configuration(self, user_params):
            user_params = BuildUserParams.from_json(user_params)

            return PluginsConfigurationRender(user_params).render()

The new class ``PluginsConfigurationRender`` will be responsible for actually
rendering each plugin. Some of its logic will be taken from ``BuildRequest``,
and ``DockJsonManipulator``. Whether functionality of ``DockJsonManipulater`` is
duplicated or reused will be clearer during implementation.

``PluginsConfigurationRender`` pseudocode::

    class PluginsConfigurationRender(object):

        def __init__(self, user_params):
            # Figure out inner template to use from user_params:
            #    <build_type>_inner:<arrangement_version>.json

        def render(self):
            # Set parameters on each plugin as needed
            return plugins_configuration

Site Customization
""""""""""""""""""

The site customization configuration file will no longer be read from the system
which created the OpenShift ``Build``, usually koji builder. Instead, this
customization file will be read from the builder image.
