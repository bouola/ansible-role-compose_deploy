# AGENTS.md

This file defines how agents should work in this repository.

These instructions apply repository-wide unless the user provides more specific
instructions.

## Intent

This is the `bouola.compose_deploy` Ansible role. It deploys Docker Compose
projects from prepared project directories to hosts that already provide Docker
Engine and Docker Compose v2.

Keep the role small, explicit, and production-friendly. It is not intended to
support every possible Compose workflow. Its public API should remain coherent,
easy to review, and safe to run on hosts that may already have running
containers and persistent data.

## Task Style

- Every task has a sentence-case `name:` in double quotes.
- Use FQCNs: `ansible.builtin.*` for built-in modules and
  `community.docker.*` only for Docker-specific operations.
- Use Ansible modules instead of `shell` or `command` whenever a suitable
  module exists. When `command` is necessary, set `changed_when` explicitly.
- Validation tasks use `ansible.builtin.assert` with an explicit multiline
  `fail_msg: >-`.
- Use descriptive loop variables such as `compose_deploy_project`, never
  `item`. Every loop sets `loop_control.loop_var` and `loop_control.label`.
- Use `ansible.builtin.stat` followed by an explicit failure task before an
  operation could overwrite or conflict with an existing path.
- Use `ansible.builtin.debug` to surface intentional no-op decisions; do not
  silently skip safety-related work.
- Keep `when` conditions explicit and readable. Add a concise comment where a
  condition is non-obvious or intentionally suppresses work.
- Keep tasks focused and easy to scan. Split a new concern into a dedicated
  task file rather than growing a task file past roughly 150 lines.

## Variables And Public API

- All public variables use the `compose_deploy_` prefix.
- Defaults are typed and truthful: each default exposed in `defaults/main.yml`
  must be implemented end-to-end.
- Use YAML booleans `true` and `false`, never `yes` or `no`.
- Repeatable inputs must be lists of dictionaries, with documented keys and
  validation before use.
- Prefer one intent-driven API. Do not introduce competing names or
  backwards-compatible aliases without an explicit user decision.
- Current supported inputs are `compose_deploy_projects` and
  `compose_deploy_volumes_dir`.

```yaml
compose_deploy_projects:
  - name: myapp
    src: files/myapp/
    dest: /opt/myapp
    mode: "0644"
    directory_mode: "0755"

compose_deploy_volumes_dir:
  - path: /srv/myapp/data
    mode: "0755"
```

## Idempotence And Safety

- A Compose project is a self-contained directory copied and deployed as-is.
  Use `project_src`; do not reconstruct an application from fragmented role
  variables.
- Never change permissions on an existing Docker-managed volume or bind-mount
  directory unless the user explicitly asks for it.
- For configured directories, inspect the current state before creation.
- Create missing directories with the requested mode.
- If a path exists and is not a directory, fail with an actionable message.
- If a directory exists with a different mode, leave it unchanged and emit a
  debug message explaining the decision.
- Do not use `recurse: true` on existing project or volume paths unless the
  user explicitly requests recursive permission normalization.
- Avoid destructive behavior hidden behind idempotence.
- Render `.j2` project files with `ansible.builtin.template` and never copy the
  raw template to the managed host.

## Task Organization

- Group tasks by purpose: prerequisites, validation, filesystem preparation,
  project synchronization, then Compose deployment.
- Keep `tasks/main.yml` as the clear entry point. Include dedicated task files
  for distinct concerns and add a short comment above each `include_tasks` that
  explains its purpose.
- Register intermediate data only when a following task consumes it.
- Avoid deeply nested Jinja expressions inline; introduce a clearly named
  variable when it improves readability.

## Task Tags

- Add tags to each independently useful task block so operators can run a
  focused part of the role without triggering unrelated work.
- Prefix every role tag with `role-compose-deploy-` to avoid collisions with
  tags defined by playbooks or other roles.
- Use only these coarse-grained tags unless a real operational workflow
  requires another one:
  - `role-compose-deploy-prerequisites` for Docker Python dependencies;
  - `role-compose-deploy-validate` for input validation;
  - `role-compose-deploy-volumes` for bind-mount directory checks and creation;
  - `role-compose-deploy-project-directories` for project root directory checks
    and creation;
  - `role-compose-deploy-project-sync` for copying files and rendering
    templates; and
  - `role-compose-deploy-deploy` for `docker compose` reconciliation.
- Keep project synchronization separate from project directory preparation. For
  example, `--tags role-compose-deploy-project-sync` must allow updating
  project files without rerunning the project root directory creation block.
- When tagging an `include_tasks`, ensure the included tasks receive the tag as
  well, using `apply.tags` or explicit task tags as appropriate.
- Do not add a tag for every individual task or module call. Tags exist for
  meaningful operational phases, not implementation details.

## Testing And Validation

Before considering a change complete, run the local checks that are available:

- `yamllint`
- `ansible-lint`
- `molecule` when Docker access is available

Scenario checks belong in Ansible `molecule/*/verify.yml` playbooks. Do not add
Python or testinfra verification tests. If the runtime scenario cannot run
because Docker access is unavailable, state that clearly in the final report.

## Documentation And Metadata

- Once the API is stable, README examples must match the implemented contract
  and use the Galaxy FQCN `bouola.compose_deploy`.
- Remove documentation for deleted variables and workflows.
- Keep examples runnable, not aspirational.
- Keep Galaxy metadata, Molecule role references, badges, and GitHub links
  aligned with the `bouola` namespace.

## When To Ask The User

Ask before an implementation changes the public contract, including:

- supported project structure;
- overwrite policy for existing project files;
- ownership handling;
- backward compatibility versus cleanup; or
- whether the role should install or manage Docker itself.

When the existing contract makes the intent clear, implement directly.
