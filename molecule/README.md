<!--
SPDX-FileCopyrightText: 2018-2025 Slavi Pantaleev
SPDX-FileCopyrightText: 2019-2022 Aaron Raimist
SPDX-FileCopyrightText: 2019-2023 MDAD project contributors
SPDX-FileCopyrightText: 2023 QEDeD
SPDX-FileCopyrightText: 2024 Fabio Bonelli
SPDX-FileCopyrightText: 2024 Nikita Chernyi
SPDX-FileCopyrightText: 2024-2026 Suguru Hirahara
SPDX-FileCopyrightText: 2026 spatterlight

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Molecule Testing

This role supports [Molecule](https://docs.ansible.com/projects/molecule/), an Ansible testing framework designed for developing and testing Ansible collections, playbooks, and roles.

## Prerequisites

To utilize Molecule you need to prepare several requirements:

- **x86** computer running one of these operating systems that make use of [systemd](https://systemd.io/):
  - **Archlinux**
  - **CentOS**, **Rocky Linux**, **AlmaLinux**, or possibly other RHEL alternatives (although your mileage may vary)
  - **Debian** (10/Buster or newer)
  - **Ubuntu** (18.04 or newer, although [20.04 may be problematic](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/ansible.md#supported-ansible-versions) if you run the Ansible playbook on it)
- `root` access on the computer which Molecule runs against
- [Ansible](http://ansible.com/) program
- [Python](https://www.python.org/)
  - Most distributions install Python by default, but some don't (e.g. Ubuntu 18.04) and require manual installation (something like `apt-get install python3`)
- [Docker](https://www.docker.com)
  - Access to Docker UNIX socket (`/var/run/docker.sock`) is required by default

## Installation

To set up the environment for using Molecule, run the command below on the terminal:

```bash
python3 -m venv ./molecule/venv
source ./molecule/venv/bin/activate
pip3 install -r ./molecule/requirements.txt
```

## Scenarios

Currently these testing scenarios are available:

### `default`

Tests a standard GitLab installation with the role's default configuration, which uses the Postgres and Redis servers bundled in the GitLab container image.

### `postgres`

Tests a standard GitLab installation with an external Postgres database (via a Unix socket), installed with [ansible-role-postgres](https://github.com/mother-of-all-self-hosting/ansible-role-postgres).

The database password deliberately contains characters (`"`, `\` and `#`) which would break the `gitlab.rb` file (which is Ruby code) if the role did not escape them correctly. The SSH port is deliberately not published, so that GitLab runs without its SSH server.

### `postgres-valkey`

Tests a standard GitLab installation with an external Postgres database (via a Unix socket) and an external Valkey data-store (via TCP), installed with [ansible-role-postgres](https://github.com/mother-of-all-self-hosting/ansible-role-postgres) and [ansible-role-valkey](https://github.com/mother-of-all-self-hosting/ansible-role-valkey). It also enables the container registry.

### What is verified

The verification does not stop at "the systemd service is active" — the unit is `Restart=always`, so a crash-looping container reports `active` too. It:

- waits for GitLab's sign-in page rather than for the unit, since on its first start GitLab needs minutes to set up its database before any of its web services come up, and then for GitLab to finish reconfiguring itself, as its last steps can restart services after the sign-in page already responds
- establishes that the API refuses unauthenticated requests and that a wrong password is rejected, so that the two checks below are able to fail in the first place
- signs in as `root` with the password the role writes into `gitlab.rb` (`gitlab_config_initial_root_password`). A GitLab running on anything but the role's configuration generates a random password instead
- asserts that the running GitLab reports the version `gitlab_version` pins and the expected edition, via `/api/v4/version`
- asserts that the bundled Postgres and Redis servers run inside the container if and only if the scenario asks for them, which proves that the database and Redis settings reached GitLab
- asserts that GitLab's SSH server runs and answers on the published port if and only if one is published, and (if enabled) that the container registry answers on its own
- creates a backup with GitLab's own backup tool (`gitlab-backup create`). With an external Postgres server, this proves that `gitlab_database_postgres_version` selects a `pg_dump` client which matches the server, as a mismatched one refuses to dump it
- asserts that `gitlab-rake gitlab:background_migrations:status`, which the role runs (and parses) before upgrading GitLab to a new minor or major version, works and still prints what the role looks for
- with an external Postgres server, asserts that the `amcheck` extension (which GitLab requires, and only a superuser may create) exists
- asserts that no Traefik labels are emitted while Traefik is disabled, and that `gitlab.rb` (which contains secrets) is only readable by `root`
- watches the services for 45 seconds to make sure none of them is quietly restarting

## Running

By default it is configured to run the scenarios on Ubuntu 26.04.

```bash
molecule test --scenario-name default
```

You can utilize other distributions by setting one to the `MOLECULE_DISTRO` environment variable:

```bash
# Ubuntu 24.04
MOLECULE_DISTRO=ubuntu2404 molecule test --scenario-name default

# Debian 13
MOLECULE_DISTRO=debian13 molecule test --scenario-name default

# Debian 12
MOLECULE_DISTRO=debian12 molecule test --scenario-name default
```

The GitLab container image is large (about 1.5 GB to download, 5.5 GB unpacked), and it is pulled from Docker Hub by the Docker daemon inside the test container. If Docker Hub rate-limits you, you can have that daemon use a registry mirror by setting the `MOLECULE_DOCKER_REGISTRY_MIRROR` environment variable:

```bash
MOLECULE_DOCKER_REGISTRY_MIRROR=https://mirror.gcr.io molecule test --scenario-name default
```
