# GitLab Container Registry Administration

> **Note:**
This feature was [introduced][ce-4040] in GitLab 8.8.

With the Docker Container Registry integrated into GitLab, every project can
have its own space to store its Docker images.

You can read more about Docker Registry at https://docs.docker.com/registry/introduction/.

---

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Enable the Container Registry](#enable-the-container-registry)
- [Container Registry domain configuration](#container-registry-domain-configuration)
    - [Configure Container Registry under an existing GitLab domain](#configure-container-registry-under-an-existing-gitlab-domain)
    - [Configure Container Registry under its own domain](#configure-container-registry-under-its-own-domain)
- [Disable Container Registry site-wide](#disable-container-registry-site-wide)
- [Disable Container Registry per project](#disable-container-registry-per-project)
- [Disable Container Registry for new projects site-wide](#disable-container-registry-for-new-projects-site-wide)
- [Container Registry storage path](#container-registry-storage-path)
- [Storage limitations](#storage-limitations)
- [Changelog](#changelog)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Enable the Container Registry

**Omnibus GitLab installations**

All you have to do is configure the domain name under which the Container
Registry will listen to. Read [#container-registry-domain-configuration](#container-registry-domain-configuration)
and pick one of the two options that fits your case.

>**Note:**
The container Registry works under HTTPS by default. Using HTTP is possible
but not recommended and out of the scope of this document.
Read the [insecure Registry documentation][docker-insecure] if you want to
implement this.

---

**Installations from source**

If you have installed GitLab from source:

1. You will have to [install Docker Registry][registry-deploy] by yourself.
1. After the installation is complete, you will have to configure the Registry's
   settings in `gitlab.yml` in order to enable it.
1. Use the sample NGINX configuration file that is found under
   [`lib/support/nginx/registry-ssl`][registry-ssl] and edit it to match the
   `host`, `port` and TLS certs paths.

The contents of `gitlab.yml` are:

```
registry:
  enabled: true
  host: registry.gitlab.example.com
  port: 5005
  api_url: http://localhost:5000/
  key: config/registry.key
  path: shared/registry
  issuer: gitlab-issuer
```

where:

| Parameter | Description |
| --------- | ----------- |
| `enabled` | `true` or `false`. Enables the Registry in GitLab. By default this is `false`. |
| `host`    | The host URL under which the Registry will run and the users will be able to use. |
| `port`    | The port under which the external Registry domain will listen on. |
| `api_url` | The internal API URL under which the Registry is exposed to. It defaults to `http://localhost:5000`. |
| `key`     | The private key location that is a pair of Registry's `rootcertbundle`. Read the [token auth configuration documentation][token-config]. |
| `path`    | This should be the same directory like specified in Registry's `rootdirectory`. Read the [storage configuration documentation][storage-config]. This path needs to be readable by the GitLab user, the web-server user and the Registry user. Read more in [#container-registry-storage-path](#container-registry-storage-path). |
| `issuer`  | This should be the same value as configured in Registry's `issuer`. Read the [token auth configuration documentation][token-config]. |

>**Note:**
GitLab does not ship with a Registry init file. Hence, [restarting GitLab][restart gitlab]
will not restart the Registry should you modify its settings. Read the upstream
documentation on how to achieve that.

## Container Registry domain configuration

There are two ways you can configure the Registry's external domain.

- Either [use the existing GitLab domain][existing-domain] where in that case
  the Registry will have to listen on a port and reuse GitLab's TLS certificate,
- or [use a completely separate domain][new-domain] with a new TLS certificate
  for that domain.

Since the container Registry requires a TLS certificate, in the end it all boils
down to how easy or pricey is to get a new one.

Please take this into consideration before configuring the Container Registry
for the first time.

### Configure Container Registry under an existing GitLab domain

If the Registry is configured to use the existing GitLab domain, you can
expose the Registry on a port so that you can reuse the existing GitLab TLS
certificate.

Assuming that the GitLab domain is `https://gitlab.example.com` and the port the
Registry is exposed to the outside world is `4567`, here is what you need to set
in `gitlab.rb` or `gitlab.yml` if you are using Omnibus GitLab or installed
GitLab from source respectively.

---

**Omnibus GitLab installations**

1. Your `/etc/gitlab/gitlab.rb` should contain the Registry URL as well as the
   path to the existing TLS certificate and key used by GitLab:

    ```ruby
    registry_external_url 'https://gitlab.example.com:4567'
    ```

    Note how the `registry_external_url` is listening on HTTPS under the
    existing GitLab URL, but on a different port.

    If your TLS certificate is not in `/etc/gitlab/ssl/gitlab.example.com.crt`
    and key not in `/etc/gitlab/ssl/gitlab.example.com.key` uncomment the lines
    below:

    ```ruby
    registry_nginx['ssl_certificate'] = "/path/to/certificate.pem"
    registry_nginx['ssl_certificate_key'] = "/path/to/certificate.key"
    ```

1. Save the file and [reconfigure GitLab][] for the changes to take effect.

---

**Installations from source**

1. Open `/home/git/gitlab/config/gitlab.yml`, find the `registry` entry and
   configure it with the following settings:

    ```
    registry:
      enabled: true
      host: gitlab.example.com
      port: 4567
    ```

1. Save the file and [restart GitLab][] for the changes to take effect.
1. Make the relevant changes in NGINX as well (domain, port, TLS certificates path).

---

Users should now be able to login to the Container Registry with their GitLab
credentials using:

```bash
docker login gitlab.example.com:4567
```

### Configure Container Registry under its own domain

If the Registry is configured to use its own domain, you will need a TLS
certificate for that specific domain (e.g., `registry.example.com`) or maybe
a wildcard certificate if hosted under a subdomain  of your existing GitLab
domain (e.g., `registry.gitlab.example.com`).

Let's assume that you want the container Registry to be accessible at
`https://registry.gitlab.example.com`.

---

**Omnibus GitLab installations**

1. Place your TLS certificate and key in
   `/etc/gitlab/ssl/registry.gitlab.example.com.crt` and
   `/etc/gitlab/ssl/registry.gitlab.example.com.key` and make sure they have
   correct permissions:

    ```bash
    chmod 600 /etc/gitlab/ssl/registry.gitlab.example.com.*
    ```

1. Once the TLS certificate is in place, edit `/etc/gitlab/gitlab.rb` with:

    ```ruby
    registry_external_url 'https://registry.gitlab.example.com'
    ```

    Note how the `registry_external_url` is listening on HTTPS.

1. Save the file and [reconfigure GitLab][] for the changes to take effect.

> **Note:**
If you have a [wildcard certificate][], you need to specify the path to the
certificate in addition to the URL, in this case `/etc/gitlab/gitlab.rb` will
look like:
>
```ruby
registry_nginx['ssl_certificate'] = "/etc/gitlab/ssl/certificate.pem"
registry_nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/certificate.key"
```

---

**Installations from source**

1. Open `/home/git/gitlab/config/gitlab.yml`, find the `registry` entry and
   configure it with the following settings:

    ```
    registry:
      enabled: true
      host: registry.gitlab.example.com
    ```

1. Save the file and [restart GitLab][] for the changes to take effect.
1. Make the relevant changes in NGINX as well (domain, port, TLS certificates path).

---

Users should now be able to login to the Container Registry using their GitLab
credentials:

```bash
docker login registry.gitlab.example.com
```

## Disable Container Registry site-wide

>**Note:**
Disabling the Registry in the Rails GitLab application as set by the following
steps, will not remove any existing Docker images. This is handled by the
Registry application itself.

**Omnibus GitLab**

1. Open `/etc/gitlab/gitlab.rb` and set `registry['enable']` to `false`:

    ```ruby
    registry['enable'] = false
    ```

1. Save the file and [reconfigure GitLab][] for the changes to take effect.

---

**Installations from source**

1. Open `/home/git/gitlab/config/gitlab.yml`, find the `registry` entry and
   set `enabled` to `false`:

    ```
    registry:
      enabled: false
    ```

1. Save the file and [restart GitLab][] for the changes to take effect.

## Disable Container Registry per project

If Registry is enabled in your GitLab instance, but you don't need it for your
project, you can disable it from your project's settings. Read the user guide
on how to achieve that.

## Disable Container Registry for new projects site-wide

If the Container Registry is enabled, then it will be available on all new
projects. To disable this function and let the owners of a project to enable
the Container Registry by themselves, follow the steps below.

---

**Omnibus GitLab installations**

1. Edit `/etc/gitlab/gitlab.rb` and add the following line:

    ```ruby
    gitlab_rails['gitlab_default_projects_features_container_registry'] = false
    ```

1. Save the file and [reconfigure GitLab][] for the changes to take effect.

---

**Installations from source**

1. Open `/home/git/gitlab/config/gitlab.yml`, find the `default_projects_features`
   entry and configure it so that `container_registry` is set to `false`:

    ```
    ## Default project features settings
    default_projects_features:
      issues: true
      merge_requests: true
      wiki: true
      snippets: false
      builds: true
      container_registry: false
    ```

1. Save the file and [restart GitLab][] for the changes to take effect.

## Container Registry storage path

To change the storage path where Docker images will be stored, follow the
steps below.

This path is accessible to:

- the user running the Container Registry daemon,
- the user running GitLab

> **Warning** You should confirm that all GitLab, Registry and web server users
have access to this directory.

---

**Omnibus GitLab installations**

The default location where images are stored in Omnibus, is
`/var/opt/gitlab/gitlab-rails/shared/registry`. To change it:

1. Edit `/etc/gitlab/gitlab.rb`:

    ```ruby
    gitlab_rails['registry_path'] = "/path/to/registry/storage"
    ```

1. Save the file and [reconfigure GitLab][] for the changes to take effect.

---

**Installations from source**

The default location where images are stored in source installations, is
`/home/git/gitlab/shared/registry`. To change it:

1. Open `/home/git/gitlab/config/gitlab.yml`, find the `registry` entry and
   change the `path` setting:

    ```
    registry:
      path: shared/registry
    ```

1. Save the file and [restart GitLab][] for the changes to take effect.

## Storage limitations

Currently, there is no storage limitation, which means a user can upload an
infinite amount of Docker images with arbitrary sizes. This setting will be
configurable in future releases.

## Changelog

**GitLab 8.8 ([source docs][8-8-docs])**

- GitLab Container Registry feature was introduced.

[reconfigure gitlab]: restart_gitlab.md#omnibus-gitlab-reconfigure
[restart gitlab]: restart_gitlab.md#installations-from-source
[wildcard certificate]: https://en.wikipedia.org/wiki/Wildcard_certificate
[ce-4040]: https://gitlab.com/gitlab-org/gitlab-ce/merge_requests/4040
[docker-insecure]: https://docs.docker.com/registry/insecure/
[registry-deploy]: https://docs.docker.com/registry/deploying/
[storage-config]: https://docs.docker.com/registry/configuration/#storage
[token-config]: https://docs.docker.com/registry/configuration/#token
[8-8-docs]: https://gitlab.com/gitlab-org/gitlab-ce/blob/8-8-stable/doc/administration/container_registry.md
[registry-ssl]: https://gitlab.com/gitlab-org/gitlab-ce/blob/master/lib/support/nginx/registry-ssl
[existing-domain]: #configure-container-registry-under-an-existing-gitlab-domain
[new-domain]: #configure-container-registry-under-its-own-domain
