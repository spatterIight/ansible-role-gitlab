<!--
SPDX-FileCopyrightText: 2020 - 2024 MDAD project contributors
SPDX-FileCopyrightText: 2020 - 2025 Slavi Pantaleev
SPDX-FileCopyrightText: 2024 - 2026 Suguru Hirahara
SPDX-FileCopyrightText: 2026 spatterlight

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up GitLab

This is an [Ansible](https://www.ansible.com/) role which installs [GitLab](https://about.gitlab.com/) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

GitLab is a complete DevOps platform: Git repository management, code reviews, issue tracking, CI/CD, a container registry and more, in a single application. See the project's [documentation](https://docs.gitlab.com/) to learn more.

The role uses GitLab's official [Omnibus container image](https://docs.gitlab.com/install/docker/) (`gitlab/gitlab-ce` or `gitlab/gitlab-ee`), which bundles all of GitLab's components, and generates its `/etc/gitlab/gitlab.rb` configuration file.

## Prerequisites

GitLab needs at least 4 GB of RAM (8 GB is recommended). Refer to [this page](https://docs.gitlab.com/install/requirements/) for details.

## Adjusting the playbook configuration

To enable GitLab, add the following configuration to your `vars.yml` file (e.g. `inventory/host_vars/mash.example.com/vars.yml` with the [MASH playbook](https://github.com/mother-of-all-self-hosting/mash-playbook)):

```yaml
########################################################################
#                                                                      #
# gitlab                                                               #
#                                                                      #
########################################################################

gitlab_enabled: true

gitlab_hostname: gitlab.example.com

########################################################################
#                                                                      #
# /gitlab                                                              #
#                                                                      #
########################################################################
```

Point the hostname's DNS records to your server.

>[!NOTE]
> GitLab can be hosted under a subpath with `gitlab_path_prefix`. Changing the hostname or the path after installation breaks the Git remote URLs of existing clones.

### Set the password for the `root` user (optional; recommended)

On its first start, GitLab creates an administrator account called `root`. To set its password (at least 8 characters, and not a commonly used one), add the following configuration to your `vars.yml` file:

```yaml
gitlab_config_initial_root_password: YOUR_PASSWORD_HERE
```

Otherwise, GitLab generates a random password, writes it to the `initial_root_password` file in the configuration directory (`gitlab_config_path`), and deletes it after 24 hours.

>[!NOTE]
> The password only takes effect on the first start. While it is set, it is stored in plain text in `gitlab.rb`, so remove it from your `vars.yml` file after signing in for the first time. Change the password in GitLab's web interface.

### Configuring the database (optional)

By default, the Postgres server bundled in the container image is used, which stores its data in `gitlab_data_path`. GitLab upgrades it to new major versions on its own terms (see [this page](https://docs.gitlab.com/omnibus/settings/database/#upgrade-packaged-postgresql-server)).

#### Using an external Postgres server

To use an external Postgres server (via TCP), add the following configuration to your `vars.yml` file:

```yaml
gitlab_database_type: postgres

gitlab_database_postgres_hostname: YOUR_POSTGRES_SERVER_HOSTNAME_HERE

gitlab_database_postgres_password: YOUR_POSTGRES_PASSWORD_HERE
```

To connect via a Unix socket instead, also add:

```yaml
gitlab_database_postgres_socket_enabled: true

# The directory containing the Postgres Unix socket on the host
gitlab_database_postgres_socket_path_host: /postgres/run
```

The password must not contain `#{`, `#$` or `#@`, which GitLab cannot handle. For other settings, see the `gitlab_database_*` variables in [`defaults/main.yml`](../defaults/main.yml).

#### Matching the Postgres version

GitLab's [backup tool](#backing-up-gitlab) uses the `pg_dump` client bundled in the container image, which cannot dump a server of a newer major version. `gitlab_database_postgres_version` selects the client matching your server, and defaults to `18` (what ansible-role-postgres installs). If your server runs another major version, set it accordingly:

```yaml
gitlab_database_postgres_version: 17
```

>[!NOTE]
> At the time of writing, GitLab 19 [officially supports](https://docs.gitlab.com/install/requirements/#postgresql) Postgres 17 only. The role is tested with Postgres 18, which GitLab works with, but which is outside that range.

#### Creating the extensions GitLab requires

GitLab requires the `pg_trgm`, `btree_gist` and `amcheck` extensions. It creates the first two on its own (so its database user needs to own the database), but only a superuser may create `amcheck`. With ansible-role-postgres, create it once by running the command below on the server:

```sh
/postgres/bin/cli-non-interactive -d gitlab -c 'CREATE EXTENSION IF NOT EXISTS amcheck;'
```

With the MASH playbook, the script is at `/mash/postgres/bin/cli-non-interactive`, and the database name is set by `gitlab_database_name`. Refer to [this page](https://docs.gitlab.com/administration/postgresql/extensions/) for details.

#### Tuning the Postgres server

GitLab lists [settings which an external Postgres server needs](https://docs.gitlab.com/administration/postgresql/tune/#required-settings-for-external-instances). ansible-role-postgres sets `shared_buffers` and `maintenance_work_mem` based on the server's memory, which is enough with 8 GB of memory.

To set `statement_timeout` and `work_mem` for GitLab's database only, run the command below on the server once:

```sh
/postgres/bin/cli-non-interactive -d gitlab \
  -c "ALTER DATABASE gitlab SET statement_timeout = '60s';" \
  -c "ALTER DATABASE gitlab SET work_mem = '8MB';"
```

`max_connections` can only be set for the whole server. GitLab asks for at least 400 (ansible-role-postgres defaults to 200):

```yaml
postgres_max_connections: 400
```

>[!IMPORTANT]
> Loading GitLab's database schema takes many locks in a single transaction. With Postgres's default settings (`max_connections=100`, `max_locks_per_transaction=64`), the installation fails with `ERROR: out of shared memory`. ansible-role-postgres's default of `max_connections=200` is enough. If you run into this error nevertheless, raise the limit:
>
> ```yaml
> postgres_process_extra_arguments_custom:
>   - "-c 'max_locks_per_transaction=128'"
> ```

### Configuring Redis (optional)

By default, the Redis server bundled in the container image is used. To use an external Redis-compatible server instead, such as [Valkey](https://valkey.io/) installed with [ansible-role-valkey](https://github.com/mother-of-all-self-hosting/ansible-role-valkey), add the following configuration to your `vars.yml` file:

```yaml
gitlab_redis_hostname: YOUR_REDIS_SERVER_HOSTNAME_HERE

# If the server runs in a container, connect GitLab to its container network
gitlab_container_additional_networks_custom:
  - YOUR_REDIS_SERVER_CONTAINER_NETWORK_HERE
```

Connecting via a Unix socket is not supported: GitLab's services run as users which only exist inside its container (e.g. `git`), and cannot access the socket directories of other containers.

If the server is shared with other services, set `gitlab_redis_database` to a database number no other service uses.

### Enabling Git over SSH (optional; recommended)

GitLab runs its own SSH server inside the container. As port 22 is usually taken by the host's own SSH server, publish it on another port, e.g. `2222`:

```yaml
gitlab_container_ssh_host_bind_port: 2222
```

GitLab then advertises clone URLs like `ssh://git@gitlab.example.com:2222/group/project.git`. Make sure that your firewall allows incoming connections on that port.

If the port is not published, the SSH server is not started, but GitLab still shows SSH clone URLs, which do not work. To hide them, set **Enabled Git access protocols** to **Only HTTP(S)** in the **Admin area** under **Settings** → **General** → **Visibility and access controls**.

#### Serving Git over SSH via Traefik

Alternatively, Traefik can forward SSH connections to GitLab with a TCP router. The SSH server then needs to be enabled explicitly, and `gitlab_config_gitlab_shell_ssh_port` set to the port clients connect to (GitLab advertises port 22 otherwise). For example, with a Traefik entrypoint on port `2222` (set up with [ansible-role-traefik](https://github.com/mother-of-all-self-hosting/ansible-role-traefik)):

```yaml
gitlab_ssh_enabled: true
gitlab_config_gitlab_shell_ssh_port: 2222

gitlab_container_labels_additional_labels_custom:
  - "traefik.tcp.routers.gitlab-ssh.rule=HostSNI(`*`)"
  - "traefik.tcp.routers.gitlab-ssh.entrypoints=gitlab-ssh"
  - "traefik.tcp.routers.gitlab-ssh.service=gitlab-ssh"
  - "traefik.tcp.services.gitlab-ssh.loadbalancer.server.port=22"

traefik_additional_entrypoints_custom:
  - name: gitlab-ssh
    port: 2222
    host_bind_port: 2222
    config: {}
```

### Enabling the container registry (optional)

To enable GitLab's [container registry](https://docs.gitlab.com/user/packages/container_registry/) on its own hostname, add the following configuration to your `vars.yml` file:

```yaml
gitlab_registry_enabled: true

gitlab_registry_hostname: registry.example.com
```

Point the hostname's DNS records to your server.

### Configuring the mailer (optional; recommended)

To have GitLab send emails (notifications, password resets) via an SMTP server, add the following configuration to your `vars.yml` file:

```yaml
gitlab_config_smtp_enabled: true
gitlab_config_smtp_address: smtp.example.com
gitlab_config_smtp_port: 587
gitlab_config_smtp_user_name: YOUR_SMTP_USERNAME_HERE
gitlab_config_smtp_password: YOUR_SMTP_PASSWORD_HERE

gitlab_config_email_from: gitlab@example.com
```

This uses STARTTLS (typically on port 587). For implicit TLS (typically on port 465), also set `gitlab_config_smtp_tls: true`. For other settings, see the `gitlab_config_smtp_*` and `gitlab_config_email_*` variables in [`defaults/main.yml`](../defaults/main.yml).

Unless SMTP is enabled, the role disables sending emails altogether (`gitlab_config_email_enabled`), as GitLab cannot send them any other way in its container.

### Reducing memory usage (optional)

The bundled Prometheus monitoring stack is disabled by default. To reduce memory usage further at the cost of performance, you can for example add:

```yaml
# Run Puma in single mode, instead of with one worker process per CPU core
gitlab_config_puma_worker_processes: 0

gitlab_config_sidekiq_concurrency: 10
```

Refer to [this page](https://docs.gitlab.com/omnibus/settings/memory_constrained_envs/) for further options.

### Adjusting Traefik configuration for large pushes and uploads (optional; recommended)

To allow long-running requests, such as pushes of large repositories or uploads over slow connections, raise Traefik's timeouts:

```yaml
########################################################################
#                                                                      #
# traefik                                                              #
#                                                                      #
########################################################################

# Your regular Traefik configuration here.

traefik_config_entrypoint_web_secure_transport_respondingTimeouts_readTimeout: 1800s
traefik_config_entrypoint_web_secure_transport_respondingTimeouts_writeTimeout: 1800s
traefik_config_entrypoint_web_secure_transport_respondingTimeouts_idleTimeout: 1800s

########################################################################
#                                                                      #
# /traefik                                                             #
#                                                                      #
########################################################################
```

### Extending the configuration

GitLab has many more settings than the role exposes as variables (see [`defaults/main.yml`](../defaults/main.yml)). To configure any of them, add Ruby code to `gitlab_configuration_extension_rb`, which is appended to the generated [`gitlab.rb`](../templates/gitlab.rb.j2) and can override the role's settings. Refer to [GitLab's configuration template](https://gitlab.com/gitlab-org/omnibus-gitlab/-/blob/master/files/gitlab-config-template/gitlab.rb.template) for the available settings.

For example, to set up [OpenID Connect](https://docs.gitlab.com/administration/auth/oidc/) as a sign-in method:

```yaml
gitlab_configuration_extension_rb: |
  gitlab_rails['omniauth_allow_single_sign_on'] = ['openid_connect']
  gitlab_rails['omniauth_block_auto_created_users'] = false
  gitlab_rails['omniauth_providers'] = [
    {
      name: 'openid_connect',
      label: 'My identity provider',
      args: {
        name: 'openid_connect',
        scope: ['openid', 'profile', 'email'],
        response_type: 'code',
        issuer: 'https://auth.example.com',
        discovery: true,
        client_auth_method: 'query',
        uid_field: 'preferred_username',
        pkce: true,
        client_options: {
          identifier: 'YOUR_CLIENT_ID_HERE',
          secret: 'YOUR_CLIENT_SECRET_HERE',
          redirect_uri: 'https://gitlab.example.com/users/auth/openid_connect/callback'
        }
      }
    }
  ]
```

## Installing

After configuring the playbook, run the installation command:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

With the MASH playbook, you can also run `just install-all` or `just setup-all` (see [`just`](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/just.md)).

GitLab reconfigures itself on every start (and migrates its database after an upgrade), so it takes a few minutes to become reachable.

## Usage

GitLab is then available at its hostname, e.g. `https://gitlab.example.com`. Log in with the username `root` and the password [set above](#set-the-password-for-the-root-user-optional-recommended) (or generated by GitLab).

>[!WARNING]
> By default, anyone can register an account, pending approval by an administrator. To disable sign-ups, go to the **Admin area** → **Settings** → **General** → **Sign-up restrictions** right after installing.

## Backing up GitLab

GitLab's state consists of its database, its data (Git repositories, uploads, etc.) and its secrets, which need to be backed up together. See [this page](https://docs.gitlab.com/administration/backup_restore/) for details.

To back up the database and the data to the `backups` directory in `gitlab_data_path`, run:

```sh
docker exec -t gitlab gitlab-backup create
```

Here and below, `gitlab` is the container name set by `gitlab_identifier` (e.g. `mash-gitlab` with the MASH playbook).

Backups older than 7 days are deleted whenever a new one is created. To change this, set `gitlab_config_backup_keep_time` in seconds (`0` keeps them forever).

With an external Postgres server, `gitlab_database_postgres_version` needs to match the server's major version (see [above](#matching-the-postgres-version)).

>[!IMPORTANT]
> The backup does not contain the configuration directory (`gitlab_config_path`), most importantly `gitlab-secrets.json`, without which the encrypted data in the database cannot be read. Back it up separately, and store it apart from the other backup.

## Upgrading GitLab

The role installs the version set by `gitlab_version` in [`defaults/main.yml`](../defaults/main.yml). When a newer version of the role pins a newer GitLab version, GitLab upgrades itself (including its database) the next time its container starts.

The role supports GitLab 19.2 and later. Upgrade older installations to 19.2 first.

>[!WARNING]
> Upgrades across several minor or major versions need to go through the "required upgrade stops" in between, one at a time. Refer to the [upgrade path tool](https://gitlab-com.gitlab.io/support/toolbox/upgrade-path/) and [this page](https://docs.gitlab.com/update/upgrade_paths/) to find them.

Each minor version of GitLab is released as its own version of the role, so the role's releases never skip a required upgrade stop. Before upgrading, the role fails (and GitLab keeps running the old version) if:

- GitLab's own upgrade check says it cannot upgrade to the new version directly. In that case, set `gitlab_version` in your `vars.yml` file to the next required upgrade stop (e.g. `gitlab_version: 19.5.4`), run the playbook, and repeat that for each stop. Then remove `gitlab_version` again, or GitLab stays at that version for good.
- before an upgrade to a new minor or major version, GitLab's batched background migrations have not finished. In that case, wait for them to finish (**Admin area** → **Monitoring** → **Background migrations**), and run the playbook again.

The role refuses to downgrade GitLab, and to switch an existing installation from the Enterprise Edition to the Community Edition, which needs [manual preparation](https://docs.gitlab.com/update/convert_to_ee/revert/) first. Once that is done, set `gitlab_upgrade_check_enabled: false` for the run which switches the edition.

## Troubleshooting

### Check the service's logs

Run `journalctl -fu gitlab` on the server (or the name of your service, e.g. `mash-gitlab`). The logs of each of GitLab's components are also in the logs directory (`gitlab_logs_path`).

### Run GitLab's own checks

```sh
docker exec -it gitlab gitlab-rake gitlab:check SANITIZE=true
```

### Reset the password of the `root` user

To reset the password of the `root` user (see [this page](https://docs.gitlab.com/security/reset_user_password/)), run:

```sh
docker exec -it gitlab gitlab-rake "gitlab:password:reset[root]"
```
