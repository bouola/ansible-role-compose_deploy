# Ansible role - compose_deploy
[![Maintainer](https://img.shields.io/badge/maintained%20by-bouola-e00000?style=flat-square)](https://github.com/bouola)
[![License](https://img.shields.io/github/license/bouola/ansible-role-compose_deploy?style=flat-square)](LICENSE)
[![Release](https://img.shields.io/github/v/release/bouola/ansible-role-compose_deploy?style=flat-square)](https://github.com/bouola/ansible-role-compose_deploy/releases)
[![Status](https://img.shields.io/github/actions/workflow/status/bouola/ansible-role-compose_deploy/ci.yml?style=flat-square&label=tests&branch=main)](https://github.com/bouola/ansible-role-compose_deploy/actions?query=workflow%3A%22CI%22)
[![Ansible version](https://img.shields.io/badge/ansible-%3E%3D2.17-black.svg?style=flat-square&logo=ansible)](https://github.com/ansible/ansible)
[![Ansible Galaxy](https://img.shields.io/badge/ansible-galaxy-black.svg?style=flat-square&logo=ansible)](https://galaxy.ansible.com/bouola/compose_deploy)


> :star: Star us on GitHub — it motivates us a lot!

Deploy Docker Compose projects from prepared directories.

## :warning: Requirements

Ansible >= 2.17

Supported platforms:

- Debian
- Ubuntu

The target host must already provide:

- Docker Engine
- Docker Compose v2 (`docker compose`)

The role also requires the `community.docker` collection.

## :zap: Installation

```bash
ansible-galaxy install bouola.compose_deploy
ansible-galaxy collection install community.docker
```

## :gear: Role variables

| Variable | Default value | Description |
|----------|---------------|-------------|
| compose_deploy_projects | [] | Compose project directories to copy to the remote host and deploy with `docker compose` |
| compose_deploy_volumes_dir | [] | Bind-mount directories to create before deploying a project |

Additionally, here is the expected structure of each variable.

### `compose_deploy_projects`

| Attribute | Default value | Description |
|-----------|---------------|-------------|
| src | None **required** | Local source directory of the Compose project. Plain files are copied as-is. Files ending with `.j2` are rendered and deployed without the `.j2` suffix |
| dest | None **required** | Destination directory on the remote host |
| name | omitted | Optional Compose project name passed to `docker compose` |
| mode | omitted | Optional file mode applied while copying project files |
| directory_mode | `0755` for newly created destination directories | Optional mode for directories created by the role |

### `compose_deploy_volumes_dir`

| Attribute | Default value | Description |
|-----------|---------------|-------------|
| path | None **required** | Bind-mount directory to create |
| mode | `0755` | Mode applied only when the directory does not already exist |

## :bulb: Behavior notes

- Existing bind-mount directories are never chmodded automatically if they already exist with another mode.
- If a configured volume path or project destination exists and is not a directory, the role fails explicitly.
- Project files ending with `.j2` are rendered with `ansible.builtin.template` and deployed without the `.j2` suffix.
- Raw `.j2` files are not copied to the target host.
- The role does not generate certificates and does not install Docker for you.

## :label: Task tags

Every operational phase has a role-scoped tag. Use these tags to run only the work required by an operational change.

| Tag | Scope |
|-----|-------|
| `role-compose-deploy-prerequisites` | Install Python dependencies required by the Compose module |
| `role-compose-deploy-validate` | Validate role inputs |
| `role-compose-deploy-volumes` | Check and create configured bind-mount directories |
| `role-compose-deploy-project-directories` | Check and create project root directories |
| `role-compose-deploy-project-sync` | Copy project files and render templates |
| `role-compose-deploy-deploy` | Reconcile projects with `docker compose` |

For example, update only the project files and templates without running the project root directory preparation phase:

```bash
ansible-playbook site.yml --tags role-compose-deploy-project-sync
```

When selecting a single phase, ensure its prerequisites are already present on the managed host. Tags do not implicitly run other phases.

## :arrows_counterclockwise: Dependencies

Required collection:

```yaml
collections:
  - community.docker
```

## :pencil2: Example Playbook

```yaml
---
- name: Converge
  hosts: all
  become: true
  vars:
    compose_deploy_volumes_dir:
      - path: /srv/flask/db
        mode: "0755"

    compose_deploy_projects:
      - name: flask
        src: files/flask/
        dest: /opt/flask
        directory_mode: "0755"

  roles:
    - bouola.compose_deploy
```

Example project structure:

```text
files/
└── flask/
    ├── docker-compose.yml
    └── nginx/
        └── nginx.conf.j2
```

## :closed_lock_with_key: [Hardening](HARDENING.md)

## :heart_eyes_cat: [Contributing](CONTRIBUTING.md)

## :copyright: [License](LICENSE)
[Mozilla Public License Version 2.0](https://www.mozilla.org/en-US/MPL/2.0/)
