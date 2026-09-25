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

Uses the role's defaults: the Postgres and Redis servers bundled in the GitLab container image.

### `postgres`

Uses an external Postgres server (via a Unix socket), installed with [ansible-role-postgres](https://github.com/mother-of-all-self-hosting/ansible-role-postgres).

The database password contains characters (`"`, `\` and `#`) which break `gitlab.rb` (Ruby code) unless the role escapes them. The SSH port is not published, so GitLab runs without its SSH server.

### `postgres-valkey`

Uses an external Postgres server (via a Unix socket) and an external Valkey server (via TCP), installed with [ansible-role-postgres](https://github.com/mother-of-all-self-hosting/ansible-role-postgres) and [ansible-role-valkey](https://github.com/mother-of-all-self-hosting/ansible-role-valkey), and enables the container registry.

### What is verified

The systemd service is `Restart=always`, so it reports `active` even while the container crash-loops. The verification therefore:

- waits for GitLab's sign-in page, and then for GitLab to finish reconfiguring itself, whose last steps can restart services
- checks that the API refuses unauthenticated requests and that a wrong password is rejected, so that the checks below can fail
- signs in as `root` with `gitlab_config_initial_root_password`, which only works if GitLab runs on the role's configuration
- checks that GitLab reports the version pinned by `gitlab_version` and the expected edition (`/api/v4/version`)
- checks that the bundled Postgres and Redis servers run if and only if the scenario uses them
- checks that GitLab's SSH server runs and answers if and only if its port is published, and that the container registry answers if enabled
- creates a backup with `gitlab-backup create`, which (with an external Postgres server) proves that `gitlab_database_postgres_version` selects a matching `pg_dump` client
- checks that `gitlab-rake gitlab:background_migrations:status`, which the role parses before an upgrade, still prints what the role looks for
- with an external Postgres server, checks that the `amcheck` extension exists
- checks that no Traefik labels are emitted while Traefik is disabled, and that `gitlab.rb` (which contains secrets) is only readable by `root`
- watches the services for 45 seconds to make sure none of them is restarting

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

The GitLab container image is large (about 1.5 GB to download, 5.5 GB unpacked). If Docker Hub rate-limits you, set `MOLECULE_DOCKER_REGISTRY_MIRROR` to have the Docker daemon inside the test container use a registry mirror:

```bash
MOLECULE_DOCKER_REGISTRY_MIRROR=https://mirror.gcr.io molecule test --scenario-name default
```
