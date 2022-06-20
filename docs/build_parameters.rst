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
as Koji, ODCS, SMTP, etc. They are also used for controlling some aspects
of the container images built, for example, distribution scope, vendor,
authoritative-registry, etc.

User parameters contain the unique information for a user's build request: git
repository, git branch, Koji target, etc.


Reactor Configuration
"""""""""""""""""""""

The environment configuration is supplied as a ``ConfigMap`` like::

    apiVersion: v1
    kind: ConfigMap
    data:
        "config.yaml": <encoded yaml>

The schema definition `config.json`_ in atomic-reactor contains a description
for each property.

Example of **REACTOR_CONFIG**:

.. code-block:: yaml

    version: 1

    remote_hosts:
        slots_dir: path/foo
        pools:
            x86_64:
                osbs-remote-hosts-1-x86_64:
                    enabled: true
                    auth: /secret-path
                    username: podman-user
                    slots: 1
                    socket_path: /run/user/2022/podman/podman.sock
                osbs-remote-hosts-2-x86_64:
                    enabled: false
                    auth: /secret-path
                    username: podman-user
                    slots: 2
                    socket_path: /run/user/2022/podman/podman.sock
            ppc64le:
                osbs-remote-hosts-1-ppc64le:
                    enabled: true
                    auth: /secret-path
                    username: podman-user
                    slots: 3
                    socket_path: /run/user/2022/podman/podman.sock

    koji:
        hub_url: https://koji.example.com/hub
        root_url: https://koji.example.com/root
        auth:
            ssl_certs_dir: /var/run/secrets/atomic-reactor/kojisecret
        use_fast_upload: false

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

    group_manifests: False

    platform_descriptors:
    - platform: x86_64
      architecture: amd64

    content_versions:
    - v2

    # Output registry (built images are pushed here).
    registry:
        url: https://container-registry.example.com/v2

    # Default source registry (base images are pulled from here)
    source_registry:
        url: https://registry.private.example.com

    # Additional source registries
    pull_registries:
    - url: https://registry.public.example.com

    registries_cfg_path: /var/run/secrets/atomic-reactor/v2-registry-dockercfg

    sources_command: "fedpkg sources"

    required_secrets:
    - kojisecret
    - odcssecret
    - v2-registry-dockercfg
    - client-config-secret

    skip_koji_check_for_base_image: False

    build_env_vars:
    - name: HTTP_PROXY
      value: "http://proxy.example.com"
    - name: HTTPS_PROXY
      value: "https://proxy.example.com"
    - name: NO_PROXY
      value: localhost,127.0.0.1

- ``registries_cfg_path``: A directory path where holds a docker
  configuration file for registry authentication. Either `.dockercfg`
  or `.dockerconfigjson` is supported. A typical configuration

User Parameters
"""""""""""""""

TBD


.. _`config.json`: https://github.com/containerbuildsystem/atomic-reactor/blob/master/atomic_reactor/schemas/config.json

