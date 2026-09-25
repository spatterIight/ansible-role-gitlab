<!--
SPDX-FileCopyrightText: 2026 spatterlight

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Updating the role for a new GitLab release

Unlike roles which ship upstream configuration files verbatim (and need them refreshed on every release), this role ships nothing from upstream. The `gitlab.rb` file it generates ([`templates/gitlab.rb.j2`](../templates/gitlab.rb.j2)) only holds the settings the role manages, and GitLab applies it on top of the defaults in its container image. For most releases, bumping `gitlab_version` is therefore all there is to it.

What the Molecule scenarios cannot catch is what a new release changes for an **existing** installation, as they only ever install GitLab from scratch: whether it's a required upgrade stop, whether it deprecates or removes a setting the role writes, and whether it changes GitLab's requirements. This checklist covers those.

## When to run this procedure

Renovate watches the `gitlab/gitlab-ce` Docker tag (see the `# renovate:` comment in [`defaults/main.yml`](../defaults/main.yml)) and opens the version-bump pull requests automatically (see [`.github/renovate.json`](../.github/renovate.json)):

- **Patch releases** are automerged once CI is green, and do not need this procedure. They never cross a required upgrade stop, and GitLab ships its security fixes as patch releases (usually twice a month).
- **Minor and major releases** get one pull request per minor version, which needs a human review. Run this procedure on each of them.

Renovate only considers image tags with the package revision `.0` (e.g. `19.4.1-ce.0`), which is what `gitlab_container_image_tag` assumes as well. GitLab very rarely republishes a version under another revision. If it does, that has to be handled by hand.

## Step 1 — Check the upgrade path

Look up the new version in GitLab's [upgrade paths](https://docs.gitlab.com/update/upgrade_paths/). Since GitLab 17.5, the required upgrade stops are the x.2, x.5, x.8 and x.11 minor versions of each major version (for GitLab 19: 19.2, 19.5, 19.8 and 19.11).

- Merge the pull requests of consecutive minor versions in order, oldest first. The autotag workflow then releases each of them as its own version of this role, so that someone updating the role one release at a time never skips a stop.
- Do not merge a pull request which would skip a required upgrade stop (e.g. because the stop's pull request was closed). Release the stop first.
- At a new major version, update the list of stops in the description in [`.github/renovate.json`](../.github/renovate.json), and the example in the "Upgrading GitLab" section of [`docs/configuring-gitlab.md`](configuring-gitlab.md).

## Step 2 — Check the settings the role writes

The role writes the settings in [`templates/gitlab.rb.j2`](../templates/gitlab.rb.j2) (and the container environment variables in [`templates/env.j2`](../templates/env.j2)). GitLab deprecates settings in `gitlab.rb` in one version, and removes them in a later major version. GitLab 19.2 moving the NGINX settings from `nginx[...]` to `gitlab_rails['nginx'][...]` is an example. A removed setting makes GitLab fail to reconfigure itself, and therefore to start.

- Read the [deprecations and removals](https://docs.gitlab.com/update/deprecations/), the upgrade notes of the new version (e.g. [those for GitLab 19](https://docs.gitlab.com/update/versions/gitlab_19_changes/)) and its release post for anything concerning the settings the role writes. At a new major version, read them with particular care.
- Diff GitLab's configuration template between the two versions (the tags of the omnibus-gitlab repository look like `19.4.1+ce.0`):

  ```sh
  git clone https://gitlab.com/gitlab-org/omnibus-gitlab.git
  cd omnibus-gitlab
  git diff 19.4.1+ce.0 19.5.0+ce.0 -- files/gitlab-config-template/gitlab.rb.template
  ```

- On an instance running the old version (such as the one the upgrade test in [Step 5](#step-5--verify) installs first), let GitLab check whether it uses any setting which the new version removes. In a Molecule instance, prefix the command with `docker exec gitlab-ubuntu2604-default`.

  ```sh
  docker exec gitlab gitlab-ctl check-config -v 19.5.0
  ```

- After upgrading that instance to the new version, check whether GitLab printed a `Deprecations:` section at the end of its reconfiguration, in the systemd journal (`journalctl -u gitlab`).

If the role starts using a setting which older versions of GitLab do not know about, raise the minimum version the role supports: in [`tasks/validate_config.yml`](../tasks/validate_config.yml), in the comment on `gitlab_version` in [`defaults/main.yml`](../defaults/main.yml), and in the "Upgrading GitLab" section of [`docs/configuring-gitlab.md`](configuring-gitlab.md).

## Step 3 — Check GitLab's requirements

Compare the following pages against what the role and its documentation say. At a new major version, these change more often than not.

- The [supported Postgres versions](https://docs.gitlab.com/install/requirements/#postgresql), and the [Postgres versions bundled](https://docs.gitlab.com/administration/package_information/postgresql_versions/) in the container image:
  - the note about Postgres versions in the "Database" section of [`docs/configuring-gitlab.md`](configuring-gitlab.md)
  - `gitlab_database_postgres_version` in [`defaults/main.yml`](../defaults/main.yml), which selects the bundled `pg_dump` and `psql` clients for backups. Its default needs to match the Postgres version ansible-role-postgres installs, and that version needs to be bundled in the image, or backups fail.
  - the version of ansible-role-postgres which the Molecule scenarios use (`molecule/*/requirements.yml`)
- The [required Postgres extensions](https://docs.gitlab.com/install/requirements/#extensions), and the [settings required for an external Postgres server](https://docs.gitlab.com/administration/postgresql/tune/#required-settings-for-external-instances): the "Configuring the database" section of [`docs/configuring-gitlab.md`](configuring-gitlab.md). An extension which only a superuser may create also needs to be created in the Molecule scenarios, via `additional_sql_queries` (see `molecule/postgres/molecule.yml`).
- The [supported Redis and Valkey versions](https://docs.gitlab.com/install/requirements/#redis-or-valkey): the version of ansible-role-valkey which the Molecule scenarios use.

## Step 4 — Check the container image

The role depends on a few things the container image's startup script and Dockerfile do. Diff them between the two versions (in the omnibus-gitlab repository cloned in [Step 2](#step-2--check-the-settings-the-role-writes)):

```sh
git diff 19.4.1+ce.0 19.5.0+ce.0 -- docker/
```

Look for changes to:

- the environment variables `GITLAB_SKIP_TAIL_LOGS` and `GITLAB_DISABLE_OPENSSH` in `docker/assets/init-container`, which [`templates/env.j2`](../templates/env.j2) sets
- the upgrade check in `docker/assets/init-container` (`gitlab-ctl upgrade-check`, and the version file `/var/opt/gitlab/gitlab-rails/VERSION` it reads), which [`tasks/install.yml`](../tasks/install.yml) mirrors before an upgrade
- the `HEALTHCHECK` in `docker/Dockerfile`, whose parameters the systemd service ([`templates/systemd/gitlab.service.j2`](../templates/systemd/gitlab.service.j2)) overrides in part
- the `VOLUME`s in `docker/Dockerfile`, which the systemd service mounts from the host

Before an upgrade to a new minor or major version, the role also parses the output of `gitlab-rake gitlab:background_migrations:status`. The Molecule scenarios assert that it's still in the format the role understands, so a change to it fails CI rather than going unnoticed.

## Step 5 — Verify

- `just prek-run-on-all` (runs ansible-lint, markdownlint, codespell, reuse and the Renovate configuration validator via [prek](https://prek.j178.dev/)) must pass.
- Run the Molecule scenarios, which install the new version from scratch (refer to [`molecule/README.md`](../molecule/README.md)):

  ```sh
  molecule test --scenario-name default
  molecule test --scenario-name postgres
  molecule test --scenario-name postgres-valkey
  ```

- Upgrade an installation of the previous version, which the Molecule scenarios do not do on their own. This also exercises the checks the role does before an upgrade:

  ```sh
  # Install the previous version, and wait for it to come up
  molecule converge --scenario-name default -- --extra-vars gitlab_version=19.4.1
  until [ "$(docker exec gitlab-ubuntu2604-default docker inspect --format '{{.State.Health.Status}}' gitlab)" = healthy ]; do sleep 10; done

  # Upgrade it to the new version (the service is only started, not restarted, by `converge`)
  molecule converge --scenario-name default
  docker exec gitlab-ubuntu2604-default systemctl restart gitlab

  molecule verify --scenario-name default
  molecule destroy --scenario-name default
  ```

- Re-check `git diff` before committing. For most releases, the changes should be limited to the version bump.
