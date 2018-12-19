# Anarcho-Tech NYC: Prosody ConverseJS [![Build Status](https://travis-ci.org/AnarchoTechNYC/ansible-role-prosody-conversejs.svg?branch=master)](https://travis-ci.org/AnarchoTechNYC/ansible-role-prosody-conversejs)

An [Ansible role](https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html) for installing the [ConverseJS](https://conversejs.org/) Web-based XMPP client running on [Prosody](https://prosody.im/). Notably, this role supports installing the Signal Protocol library for [OMEMO](https://en.wikipedia.org/wiki/OMEMO) support. This role's primary purpose is to provide a completely self-hosted and feature-complete ConverseJS instance running on Prosody.

If you are comfortable loading ConverseJS from a CDN, and are familiar with the [security considerations (and privacy risks)](https://conversejs.org/docs/html/security.html) that doing so implies, you do not need to use this role. Instead, you can merely configure Prosody to load ConverseJS automatically. If you wish to serve ConverseJS from your own infrastructure, this role simplifies the process of downloading the ConverseJS sources, and verifying the integrity of your ConverseJS installation.

## Configuring Prosody for ConverseJS

This [Ansible role depends](https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html#role-dependencies) on the [Anarcho-Tech NYC Prosody role](https://github.com/AnarchoTechNYC/ansible-role-prosody). You must first configure Prosody appropriately for ConverseJS using that role's variables before applying this role. An example for ConverseJS is provided in [the Prosody role's README.md file](https://github.com/AnarchoTechNYC/ansible-role-prosody/blob/master/README.md#prosody-server-configuration), and is reprinted here:

1. Simple MUC-enabled server using the [ConverseJS Web-based chat front-end](https://conversejs.org/) served via both HTTP and HTTPS on their alternate ports (`8080` and `8443`), using the server root as the ConverseJS endpoint, with in-band user registration enabled:
    ```yml
    prosody_plugins_src_base_url: https://hg.prosody.im/prosody-modules/raw-file/
    prosody_plugins:
      - name: conversejs
        version: tip
    prosody_config:
      plugins_paths:
        - /usr/local/lib/prosody/modules
      modules_enabled:
        - saslauth
        - tls
        - disco
        - register
        - pep
        - roster
        - carbons
        - vcard # Optional, but needed for ConverseJS avatar support.
      allow_registration: true
      pidfile: "{{ prosody_server_run_dir }}/prosody.pid"
      authentication: internal_hashed
      http_ports:
        - 8080
      https_ports:
        - 8443
      VirtualHosts:
        - domain: example.com
          modules_enabled:
            - conversejs
          http_paths:
            conversejs: "/"
          conversejs_options:
            view_mode: fullscreen
          conversejs_tags:
            # This script provides Signal Protocol support (needed for OMEMO).
            - '<script src="https://cdn.conversejs.org/3rdparty/libsignal-protocol.min.js"></script>'
          Components:
            - hostname: conference.example.com
              plugin: muc
              options:
                restrict_room_creation: local
    prosody_users:
      - jid: alice@example.com
        password: password
      - jid: bob@example.com
        password: password
    ```
    The above configuration will ensure that `alice@example.com` can log in via the ConverseJS Web front-end at `http://example.com:8080/` with their initial account password of `password`, as well as providing a registration link for new users.

As you can see from the above configuration, an Internet connection is required to load ConverseJS from the `cdn.conversejs.org` content delivery network. For servers on isolated networks, or in the case of an ISP or DNS outage, this will cause ConverseJS to become unavailable and fail to load. We can offer improved security by using this role to install the necessary JavaScript, CSS, and other Web-based assets to our own server and [configuring Prosody to serve those assets](#configuring-prosody-to-self-host-conversejs) instead of fetching them from a CDN on each page load.

## Role defaults

This role provides the following default variables:

* `conversejs_files_dir`: Directory to install the ConverseJS files into. Defaults to `{{ prosody_http_files_dir }}/converse.js`.
* `conversejs_libsignal`: Dictionary describing the Signal Protocol library to install. The keys of this dictionary are:
    * `path`: Path at which to download the JavaScript version of the Signal Protocol library. Defaults to `{{ prosody_http_files_dir }}/libsignal-protocol.js`
    * `version`: Git tag, branch, or commit hash of the Signal Protocol library to download. Defaults to `master`.
    * `checksum`: Optional digest in `<algorithm>:<digest>` value (same as [`get_url`](https://docs.ansible.com/ansible/latest/modules/get_url_module.html)'s `checksum` parameter) to compare the downloaded library file against.

## Configuring Prosody to self-host ConverseJS

All that is required to convert [the above Prosody configuration that uses the ConverseJS CDN](#configuring-prosody-for-conversejs) to a self-hosted version is:

1. Add the following Prosody configuration options to the `VirtualHost` dictionary where the other ConverseJS options reside:
    ```yml
    conversejs_script: http://example.com:8443/files/converse.js/dist/converse.js
    conversejs_css: http://example.com:8443/files/converse.js/css/converse.css
    ```
1. Change the `conversejs_tags` list item loading the Signal Protocol library to the one downloaded by this role. For example:
    ```yml
    conversejs_tags:
    # This script provides Signal Protocol support (needed for OMEMO).
    - '<script src="http://example.com:8443/files/libsignal-protocol.js"></script>'
    ```

The above changes will serve ConverseJS from the `example.com` server's alternate HTTPS port (`8443`).
