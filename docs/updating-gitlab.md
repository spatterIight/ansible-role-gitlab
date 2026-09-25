<!--
SPDX-FileCopyrightText: 2026 spatterlight

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Updating the role for a new GitLab release

The role's [`gitlab.rb`](../templates/gitlab.rb.j2) only holds the settings the role manages, on top of the defaults in the container image. For most releases, bumping `gitlab_version` is all there is to it.

The Molecule scenarios only install GitLab from scratch, so this checklist covers what they cannot catch: required upgrade stops, deprecated or removed settings, and changed requirements.

## When to run this procedure

Renovate watches the `gitlab/gitlab-ce` Docker tag and opens the version-bump pull requests (see [`.github/renovate.json`](../.github/renovate.json)):

- **Patch releases** are automerged once CI is green. They never cross a required upgrade stop.
- **Minor and major releases** get one pull request per minor version. Run this procedure on each of them.

Renovate only considers image tags with the package revision `.0` (e.g. `19.4.1-ce.0`), as does `gitlab_container_image_tag`. A version republished under another revision has to be handled by hand.

## Step 1 — Check the upgrade path

Look up the new version in GitLab's [upgrade paths](https://docs.gitlab.com/update/upgrade_paths/). The required upgrade stops are the x.2, x.5, x.8 and x.11 minor versions of each major version.

- Merge the pull requests of consecutive minor versions in order, oldest first, so that each is released as its own version of the role.
- Do not merge a pull request which would skip a required upgrade stop. Release the stop first.
- At a new major version, update the list of stops in [`.github/renovate.json`](../.github/renovate.json) and the example in the "Upgrading GitLab" section of [`docs/configuring-gitlab.md`](configuring-gitlab.md).

## Step 2 — Check the settings the role writes

The role writes the settings in [`templates/gitlab.rb.j2`](../templates/gitlab.rb.j2) and the environment variables in [`templates/env.j2`](../templates/env.j2). A setting which GitLab has removed makes it fail to reconfigure itself, and therefore to start.

- Read the [deprecations and removals](https://docs.gitlab.com/update/deprecations/), the upgrade notes of the new version (e.g. [those for GitLab 19](https://docs.gitlab.com/update/versions/gitlab_19_changes/)) and its release post.
- Diff GitLab's configuration template between the two versions:

  ```sh
  git clone https://gitlab.com/gitlab-org/omnibus-gitlab.git
  cd omnibus-gitlab
  git diff 19.4.1+ce.0 19.5.0+ce.0 -- files/gitlab-config-template/gitlab.rb.template
  ```

- On an instance running the old version (such as the one from [Step 5](#step-5--verify)), check for settings which the new version removes. In a Molecule instance, prefix the command with `docker exec gitlab-ubuntu2604-default`.

  ```sh
  docker exec gitlab gitlab-ctl check-config -v 19.5.0
  ```

- After upgrading that instance, check the end of its reconfiguration in `journalctl -u gitlab` for a `Deprecations:` section.

If the role starts using a setting which older versions of GitLab do not know about, raise the minimum supported version in [`tasks/validate_config.yml`](../tasks/validate_config.yml), in the comment on `gitlab_version` in [`defaults/main.yml`](../defaults/main.yml), and in the "Upgrading GitLab" section of [`docs/configuring-gitlab.md`](configuring-gitlab.md).

## Step 3 — Check GitLab's requirements

Compare the following pages against the role and its documentation:

- The [supported Postgres versions](https://docs.gitlab.com/install/requirements/#postgresql) and the [Postgres versions bundled](https://docs.gitlab.com/administration/package_information/postgresql_versions/) in the container image:
  - the "Matching the Postgres version" section of [`docs/configuring-gitlab.md`](configuring-gitlab.md)
  - `gitlab_database_postgres_version` in [`defaults/main.yml`](../defaults/main.yml). It needs to match the Postgres version ansible-role-postgres installs, and that version needs to be bundled in the image, or backups fail.
  - the version of ansible-role-postgres in `molecule/*/requirements.yml`
- The [required Postgres extensions](https://docs.gitlab.com/install/requirements/#extensions) and the [settings required for an external Postgres server](https://docs.gitlab.com/administration/postgresql/tune/#required-settings-for-external-instances): the "Using an external Postgres server" section of [`docs/configuring-gitlab.md`](configuring-gitlab.md). An extension which only a superuser may create also needs to be created in the Molecule scenarios via `additional_sql_queries` (see `molecule/postgres/molecule.yml`).
- The [supported Redis and Valkey versions](https://docs.gitlab.com/install/requirements/#redis-or-valkey): the version of ansible-role-valkey in `molecule/postgres-valkey/requirements.yml`.

## Step 4 — Check the container image

Diff the container image's files between the two versions:

```sh
git diff 19.4.1+ce.0 19.5.0+ce.0 -- docker/
```

Look for changes to:

- the environment variables `GITLAB_SKIP_TAIL_LOGS` and `GITLAB_DISABLE_OPENSSH` in `docker/assets/init-container`, which [`templates/env.j2`](../templates/env.j2) sets
- the upgrade check in `docker/assets/init-container` (`gitlab-ctl upgrade-check` and the version file `/var/opt/gitlab/gitlab-rails/VERSION`), which [`tasks/install.yml`](../tasks/install.yml) mirrors
- the `HEALTHCHECK` in `docker/Dockerfile`, which [`templates/systemd/gitlab.service.j2`](../templates/systemd/gitlab.service.j2) partly overrides
- the `VOLUME`s in `docker/Dockerfile`, which the systemd service mounts from the host

The Molecule scenarios check that the output of `gitlab-rake gitlab:background_migrations:status`, which the role parses before an upgrade, is still in the expected format.

## Step 5 — Verify

- `just prek-run-on-all` must pass.
- Run the Molecule scenarios (see [`molecule/README.md`](../molecule/README.md)):

  ```sh
  molecule test --scenario-name default
  molecule test --scenario-name postgres
  molecule test --scenario-name postgres-valkey
  ```

- Upgrade an installation of the previous version, which also exercises the role's pre-upgrade checks:

  ```sh
  # Install the previous version, and wait for it to come up
  molecule converge --scenario-name default -- --extra-vars gitlab_version=19.4.1
  until [ "$(docker exec gitlab-ubuntu2604-default docker inspect --format '{{.State.Health.Status}}' gitlab)" = healthy ]; do sleep 10; done

  # Upgrade it (`converge` only starts the service, it does not restart it)
  molecule converge --scenario-name default
  docker exec gitlab-ubuntu2604-default systemctl restart gitlab

  molecule verify --scenario-name default
  molecule destroy --scenario-name default
  ```
