<!--
SPDX-FileCopyrightText: 2020 - 2024 MDAD project contributors
SPDX-FileCopyrightText: 2020 - 2025 Slavi Pantaleev
SPDX-FileCopyrightText: 2024 - 2026 Suguru Hirahara
SPDX-FileCopyrightText: 2026 spatterlight

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up GitLab

This is an [Ansible](https://www.ansible.com/) role which installs [GitLab](https://about.gitlab.com/) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

GitLab is a complete DevOps platform: Git repository management, code reviews, issue tracking, CI/CD, a container registry and more, in a single application.

See the project's [documentation](https://docs.gitlab.com/) to learn what GitLab does and why it might be useful to you.

This role uses GitLab's official [Omnibus container image](https://docs.gitlab.com/install/docker/) (`gitlab/gitlab-ce`, or `gitlab/gitlab-ee`). The image bundles all of GitLab's components (Puma, Sidekiq, Gitaly, Workhorse, nginx, an OpenSSH server, and optionally PostgreSQL and Redis) and is configured via a `/etc/gitlab/gitlab.rb` file, which this role generates.

## Prerequisites

### System requirements

GitLab is considerably more resource-intensive than most self-hosted services. It needs at least 4 GB of RAM (8 GB is recommended), and a few GB of disk space before any repositories are stored. Refer to [this page](https://docs.gitlab.com/install/requirements/) for details.

### Database

GitLab requires a [Postgres](https://www.postgresql.org/) database. By default, the Postgres server bundled in the GitLab container image is used. Alternatively, an external Postgres server can be used, such as one installed with [ansible-role-postgres](https://github.com/mother-of-all-self-hosting/ansible-role-postgres). See [below](#using-an-external-postgres-server) for details.

>[!NOTE]
> GitLab supports only a specific range of Postgres versions. Refer to [this page](https://docs.gitlab.com/install/requirements/#postgresql) for the versions supported by the GitLab version you install. At the time of writing, GitLab 19 officially supports Postgres 17 only.
>
> For an external Postgres server, this role nevertheless defaults to (and is tested with) Postgres 18, which is what ansible-role-postgres installs. GitLab 19.3 and later bundle Postgres 18 themselves (as an option for new installations), and GitLab works with it, but be aware that it is outside the range GitLab officially supports. To stay within that range, use Postgres 17, and set `gitlab_database_postgres_version: 17` (see [below](#matching-the-postgres-version)).

## Adjusting the playbook configuration

To enable GitLab with this role, add the following configuration to your `vars.yml` file.

**Note**: the path should be something like `inventory/host_vars/mash.example.com/vars.yml` if you use the [MASH Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook).

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

After adjusting the hostname, make sure to adjust your DNS records to point the domain to your server.

>[!NOTE]
> Hosting GitLab under a subpath (by configuring the `gitlab_path_prefix` variable) is supported, but changing the hostname or the path after the initial installation requires everyone to update the Git remote URLs of their clones.

### Set the password for the `root` user (optional; recommended)

On its first start, GitLab creates an administrator account called `root`. To set its password, add the following configuration to your `vars.yml` file:

```yaml
gitlab_config_initial_root_password: YOUR_PASSWORD_HERE
```

The password needs to be at least 8 characters long, and must not be a commonly used one, or GitLab refuses to create the account with it.

If you do not set it, GitLab generates a random password and writes it to the `initial_root_password` file in the configuration directory (`gitlab_config_path`, e.g. `/mash/gitlab/config` with the MASH playbook) on the server, which it deletes after 24 hours.

>[!NOTE]
> Changing this value has no effect once GitLab has been installed. To change the password afterwards, use GitLab's web interface.
>
> As long as it is set, the password is stored in plain text in the `gitlab.rb` file in the configuration directory. Once you have signed in for the first time, you can remove it from your `vars.yml` file, which removes it from there on the next run of the playbook (and restarts GitLab).

### Configuring the database (optional)

By default, the Postgres server bundled in the GitLab container image is used. It stores its data in the `postgresql` directory in `gitlab_data_path`.

>[!WARNING]
> GitLab upgrades the bundled Postgres server to a new major version on its own terms. Refer to [this page](https://docs.gitlab.com/omnibus/settings/database/#upgrade-packaged-postgresql-server) for details.

#### Using an external Postgres server

To use an external Postgres server (via TCP) instead, add the following configuration to your `vars.yml` file:

```yaml
gitlab_database_type: postgres

gitlab_database_postgres_hostname: YOUR_POSTGRES_SERVER_HOSTNAME_HERE

gitlab_database_postgres_password: YOUR_POSTGRES_PASSWORD_HERE
```

For other settings, check variables such as `gitlab_database_*` on [`defaults/main.yml`](../defaults/main.yml).

>[!NOTE]
> GitLab cannot handle a database password which contains `#{`, `#$` or `#@`, so the role refuses to use one.

To connect to the Postgres server via a Unix socket instead, add the following configuration to your `vars.yml` file:

```yaml
gitlab_database_postgres_socket_enabled: true

# Specify the path to the directory containing the Postgres Unix socket on the host (bind-mount source)
gitlab_database_postgres_socket_path_host: /postgres/run
```

#### Matching the Postgres version

GitLab's [backup tool](#backing-up-gitlab) dumps the database with the `pg_dump` client which is bundled in the GitLab container image, and `pg_dump` refuses to dump a server of a newer major version than its own. The role therefore tells GitLab which major version your Postgres server runs, so that it uses the matching client. It defaults to `18`, which is what ansible-role-postgres installs. If your server runs another major version, add the following configuration to your `vars.yml` file:

```yaml
gitlab_database_postgres_version: 17
```

#### Creating the extensions GitLab requires

GitLab requires the Postgres extensions `pg_trgm`, `btree_gist` and `amcheck` in its database. It creates `pg_trgm` and `btree_gist` on its own (which is why the database user needs to own the database), but only a superuser may create `amcheck`, so it needs to be created once by hand.

With ansible-role-postgres, run the command below on the server. With the MASH playbook, the script is at `/mash/postgres/bin/cli-non-interactive`, and the database name is whatever `gitlab_database_name` is set to.

```sh
/postgres/bin/cli-non-interactive -d gitlab -c 'CREATE EXTENSION IF NOT EXISTS amcheck;'
```

Refer to [this page](https://docs.gitlab.com/administration/postgresql/extensions/) for details.

#### Tuning the Postgres server

GitLab lists [settings which an external Postgres server needs](https://docs.gitlab.com/administration/postgresql/tune/#required-settings-for-external-instances). ansible-role-postgres tunes `shared_buffers` and `maintenance_work_mem` on its own, based on the server's memory, which is enough on a server with the 8 GB of memory GitLab recommends.

`statement_timeout` and `work_mem` can be set for GitLab's database alone, which leaves any other services that use the same Postgres server unaffected. With ansible-role-postgres, run the command below on the server once (with the same adjustments for the MASH playbook as above):

```sh
/postgres/bin/cli-non-interactive -d gitlab \
  -c "ALTER DATABASE gitlab SET statement_timeout = '60s';" \
  -c "ALTER DATABASE gitlab SET work_mem = '8MB';"
```

`max_connections`, for which GitLab asks for at least 400, can only be set for the whole server. With ansible-role-postgres (which defaults to 200), add the following configuration to your `vars.yml` file:

```yaml
postgres_max_connections: 400
```

>[!IMPORTANT]
> Loading GitLab's database schema takes a lot of locks in a single transaction. With a Postgres server's default settings (`max_connections=100`, `max_locks_per_transaction=64`), the initial installation fails with `ERROR: out of shared memory` / `HINT: You might need to increase "max_locks_per_transaction"`.
>
> ansible-role-postgres configures `max_connections=200` by default, which is enough. If you run into this error nevertheless, raise the limit, for example with ansible-role-postgres:
>
> ```yaml
> postgres_process_extra_arguments_custom:
>   - "-c 'max_locks_per_transaction=128'"
> ```

### Configuring Redis (optional)

GitLab also requires a [Redis](https://redis.io/)-compatible data store. By default, the Redis server bundled in the GitLab container image is used.

To use an external server instead, such as [Valkey](https://valkey.io/) installed with [ansible-role-valkey](https://github.com/mother-of-all-self-hosting/ansible-role-valkey), add the following configuration to your `vars.yml` file:

```yaml
gitlab_redis_hostname: YOUR_REDIS_SERVER_HOSTNAME_HERE

# If the server runs in a container on another container network (e.g. the one of ansible-role-valkey),
# connect the GitLab container to that network as well.
gitlab_container_additional_networks_custom:
  - YOUR_REDIS_SERVER_CONTAINER_NETWORK_HERE
```

>[!NOTE]
> Connecting to the server via a Unix socket is not supported. GitLab's services run as users which only exist inside its container (e.g. `git`, with the UID 998), which cannot access the socket directories of other containers (e.g. the one of ansible-role-valkey, which is only accessible to its own user).

If the server is shared with other services, consider setting `gitlab_redis_database` to a database number that is not used by any other service.

### Enabling Git over SSH (optional; recommended)

GitLab lets people clone and push to repositories over HTTPS and SSH. For SSH, it runs its own SSH server inside the container, which needs to be published on a port on the host.

As port 22 is usually taken by the host's own SSH server, another port has to be chosen. To publish it on port `2222`, add the following configuration to your `vars.yml` file:

```yaml
gitlab_container_ssh_host_bind_port: 2222
```

GitLab then advertises clone URLs like `ssh://git@gitlab.example.com:2222/group/project.git`.

Make sure that your firewall allows incoming connections on that port.

If you do not publish the port, the SSH server inside the container is not started. GitLab still shows SSH clone URLs in its web interface though, which do not work. To hide them, set **Enabled Git access protocols** to **Only HTTP(S)** in the **Admin area** under **Settings** → **General** → **Visibility and access controls**.

#### Serving Git over SSH via Traefik

Instead of publishing the port, you can have Traefik forward SSH connections to GitLab with a TCP router. In that case, the SSH server has to be enabled explicitly, and GitLab has to be told which port clients connect to, as it advertises port 22 in its clone URLs otherwise. For example, with Traefik listening for SSH connections on port `2222` (set up with [ansible-role-traefik](https://github.com/mother-of-all-self-hosting/ansible-role-traefik) here):

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

GitLab includes a [container registry](https://docs.gitlab.com/user/packages/container_registry/), which stores container images alongside the projects that build them. It is served on its own hostname.

To enable it, add the following configuration to your `vars.yml` file:

```yaml
gitlab_registry_enabled: true

gitlab_registry_hostname: registry.example.com
```

After adjusting the hostname, make sure to adjust your DNS records to point the domain to your server.

### Configuring the mailer (optional; recommended)

GitLab sends emails to notify people about activity and to let them reset their passwords. To have it send them via an SMTP server, add the following configuration to your `vars.yml` file:

```yaml
gitlab_config_smtp_enabled: true
gitlab_config_smtp_address: smtp.example.com
gitlab_config_smtp_port: 587
gitlab_config_smtp_user_name: YOUR_SMTP_USERNAME_HERE
gitlab_config_smtp_password: YOUR_SMTP_PASSWORD_HERE

gitlab_config_email_from: gitlab@example.com
```

This connects to the SMTP server with STARTTLS (typically on port 587). If your SMTP server uses implicit TLS instead (typically on port 465), also add `gitlab_config_smtp_tls: true` (which disables STARTTLS, as the two cannot be combined).

For other settings, check variables such as `gitlab_config_smtp_*` and `gitlab_config_email_*` on [`defaults/main.yml`](../defaults/main.yml).

>[!NOTE]
> GitLab can only send emails via SMTP in its container. Unless SMTP is enabled, the role therefore disables sending emails altogether (see `gitlab_config_email_enabled`), so that GitLab does not keep trying (and failing) to send them.

### Reducing memory usage (optional)

The bundled Prometheus monitoring stack is disabled by default, as it uses a considerable amount of memory. On a server with little memory, you can reduce GitLab's memory usage further at the cost of performance, for example with the following configuration:

```yaml
# Run Puma in single mode, instead of with one worker process per CPU core
gitlab_config_puma_worker_processes: 0

gitlab_config_sidekiq_concurrency: 10
```

Refer to [this page](https://docs.gitlab.com/omnibus/settings/memory_constrained_envs/) for details and further options.

### Adjusting Traefik configuration for large pushes and uploads (optional; recommended)

We recommend **adjusting your Traefik configuration** to allow long-running requests, which may be necessary for pushing large repositories, or uploading large files over slow connections.

```yaml
########################################################################
#                                                                      #
# traefik                                                              #
#                                                                      #
########################################################################

# Your regular Traefik configuration here.

# Increase some timeouts to allow for long-running requests to GitLab.
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

There are some additional things you may wish to configure about the component.

Take a look at:

- [`defaults/main.yml`](../defaults/main.yml) for some variables that you can customize via your `vars.yml` file
- [`templates/gitlab.rb.j2`](../templates/gitlab.rb.j2) for the `gitlab.rb` configuration file which this role generates

GitLab has many more settings than this role exposes as variables. To configure any of them, add Ruby code to the `gitlab_configuration_extension_rb` variable, which is appended to `gitlab.rb` (and can therefore also override any setting the role configures). Refer to [GitLab's configuration template](https://gitlab.com/gitlab-org/omnibus-gitlab/-/blob/master/files/gitlab-config-template/gitlab.rb.template) for the available settings.

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

After configuring the playbook, run the installation command of your playbook as below:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

If you use the MASH playbook, the shortcut commands with the [`just` program](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/just.md) are also available: `just install-all` or `just setup-all`

>[!NOTE]
> Every time it starts, GitLab reconfigures itself and (after an upgrade) migrates its database before serving any requests. This takes a few minutes, during which GitLab is not reachable yet.

## Usage

After running the command for installation, GitLab becomes available at the specified hostname like `https://gitlab.example.com`.

To get started, open the URL with a web browser, and log in to the instance with the username `root` and the password you configured with `gitlab_config_initial_root_password` (or the one GitLab generated, see [above](#set-the-password-for-the-root-user-optional-recommended)).

>[!WARNING]
> By default, GitLab allows anyone to register an account, though new accounts need to be approved by an administrator. If you do not want this, disable sign-ups in the **Admin area** under **Settings** → **General** → **Sign-up restrictions** right after installing.

## Backing up GitLab

A GitLab instance's state consists of its database, its data (Git repositories, uploads, etc.) and its secrets, all of which need to be backed up together. See [this page](https://docs.gitlab.com/administration/backup_restore/) for details.

To create a backup of the database and the data with GitLab's own backup tool, run the command below on the server. The backup is written to the `backups` directory in the data directory (`gitlab_data_path`).

```sh
docker exec -t gitlab gitlab-backup create
```

Here and below, `gitlab` is the name of the container, which is set by `gitlab_identifier` (e.g. `mash-gitlab` with the MASH playbook).

Backups older than 7 days are deleted whenever a new one is created. To keep them for longer (or forever, with `0`), set `gitlab_config_backup_keep_time` to the number of seconds to keep them for.

With an external Postgres server, the backup tool can only dump the database if `gitlab_database_postgres_version` matches the server's major version (see [above](#matching-the-postgres-version)).

>[!IMPORTANT]
> This backup does not contain the configuration and secrets in the configuration directory (`gitlab_config_path`; most importantly `gitlab-secrets.json`), without which the encrypted data in the database cannot be read. Back them up separately, and store them apart from the other backup.

## Upgrading GitLab

This role installs the GitLab version specified in `gitlab_version` on [`defaults/main.yml`](../defaults/main.yml). When a newer version of this role pins a newer GitLab version, GitLab upgrades itself (including its database) the next time its container is started.

This role supports GitLab 19.2 and later only. To manage an existing installation of an older version with this role, upgrade it to 19.2 or later first.

>[!WARNING]
> GitLab cannot be upgraded from any version to any other version directly. Upgrades across several minor or major versions need to go through the "required upgrade stops" in between, one at a time. Refer to the [upgrade path tool](https://gitlab-com.gitlab.io/support/toolbox/upgrade-path/) and [this page](https://docs.gitlab.com/update/upgrade_paths/) to find them.

Each new minor version of GitLab is released as its own version of this role, so that the role's releases never skip a required upgrade stop. Before installing a new version, the role checks:

- with GitLab's own tool, whether GitLab can be upgraded to it directly from the version it runs. If it cannot, the role fails (and GitLab keeps running the version it runs). In that case, set `gitlab_version` in your `vars.yml` file to the next required upgrade stop (for example `gitlab_version: 19.5.4`), run the playbook, and repeat that for each stop in turn. Once you have caught up, remove `gitlab_version` from your `vars.yml` file again, as it would otherwise keep GitLab at that version for good.
- before an upgrade to a new minor or major version, whether GitLab's batched background migrations have finished (as long as GitLab is running). GitLab requires this, as the database migrations of the new version may depend on them. If they have not finished yet, the role fails. In that case, wait for them to finish (see **Admin area** → **Monitoring** → **Background migrations**), and run the playbook again. This is also why you should wait for them to finish at each required upgrade stop.

Downgrading GitLab is not supported, and the role refuses to do it. It also refuses to switch an existing installation from the Enterprise Edition to the Community Edition, which needs [manual preparation](https://docs.gitlab.com/update/convert_to_ee/revert/) first. Once that is done, set `gitlab_upgrade_check_enabled: false` for the one run which switches the edition.

## Troubleshooting

### Check the service's logs

You can find the logs in [systemd-journald](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html) by logging in to the server with SSH and running `journalctl -fu gitlab` (or how you/your playbook named the service, e.g. `mash-gitlab`).

The logs of each of GitLab's components are also available in the logs directory (`gitlab_logs_path`) on the server.

### Run GitLab's own checks

GitLab can check its configuration and environment on its own:

```sh
docker exec -it gitlab gitlab-rake gitlab:check SANITIZE=true
```

### Reset the password of the `root` user

If you lose the password of the `root` user, you can reset it with the command below. Refer to [this page](https://docs.gitlab.com/security/reset_user_password/) for details.

```sh
docker exec -it gitlab gitlab-rake "gitlab:password:reset[root]"
```
