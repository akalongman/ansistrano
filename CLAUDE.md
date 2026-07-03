# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A thin Ansible wrapper (Composer package `longman/ansistrano`) around the community
[`ansistrano.deploy`](https://github.com/ansistrano/deploy) and `ansistrano.rollback`
Galaxy roles. Those roles provide Capistrano-style, zero-downtime, release-based
deployment (each deploy lands in `releases/<timestamp>/`, then a `current` symlink
is flipped). This repo adds no deployment engine of its own. It only injects
PHP/Laravel-specific steps into Ansistrano's lifecycle hooks.

The package is meant to be installed inside a PHP project (typically under
`vendor/longman/ansistrano/`) and run from there. That placement is baked into
`vars/defaults.yml`: `ansistrano_deploy_from: "{{ playbook_dir }}/../../.."` walks
up three levels to the consuming project root, which is what gets deployed.

## Commands

Roles are pinned in `requirements.yml` (deploy 3.14.0, rollback 3.1.0). Install them first.

```bash
ansible-galaxy install -r requirements.yml            # install pinned roles
ansible-galaxy install --force ansistrano.deploy ansistrano.rollback   # per README, unpinned

ansible-playbook -i example/hosts.ini deploy.yml      # deploy
ansible-playbook -i example/hosts.ini rollback.yml    # roll back one release
ansible-playbook -i example/hosts.ini cleanup.yml     # tear down a dynamic deployment
```

There is no test suite. The practical equivalents for iterating safely on a single
change are Ansible's own selectors, run against a real inventory:

```bash
ansible-playbook -i inv deploy.yml --syntax-check       # parse only
ansible-playbook -i inv deploy.yml --check              # dry run
ansible-playbook -i inv deploy.yml --tags pre           # run one lifecycle phase (see Tags)
ansible-playbook -i inv deploy.yml --start-at-task "APP | Restart Horizon"
ansible-playbook -i inv deploy.yml --limit task -vvv    # one host group, verbose
```

## Architecture

### The hook-dispatch pattern

Ansistrano exposes hook points around its lifecycle (setup, update_code, symlink,
cleanup). `vars/defaults.yml` binds this repo's dispatcher files to those hooks via
the `ansistrano_*_tasks_file` variables. Each dispatcher in `config/` is not a task
itself; it is a list of `include_tasks` that conditionally pull in reusable task
files from `config/tasks/`.

| Dispatcher (`config/`) | Runs | Typical contents |
|---|---|---|
| `before_setup.yml` | before the release dir exists | dynamic infra (DNS, folders, mounts, nginx) |
| `before_update_code.yml` | before code sync | sync storage content |
| `before_symlink.yml` | new release built, not yet live | permissions, cache warm (`optimize_pre`), self-diagnosis, doctrine |
| `after_symlink.yml` | new release is now live | opcache reset, restart php-fpm/queue/horizon/supervisor, cron, Slack |
| `after_cleanup.yml` | after old releases pruned | copyright notice |
| `on_failure.yml` | any task fails | Slack danger notification (rollback include is present but commented out) |

The top-level playbooks are `deploy.yml`, `rollback.yml`, and `cleanup.yml`.
`deploy.yml` and `cleanup.yml` set `any_errors_fatal: true`, so any host failing
aborts the whole run.

### Feature-flag toggles

Every optional task is gated behind an `app_*` boolean checked with the idiom
`when: "app_foo is defined and app_foo|bool == true"`. Toggles are set per host
group in the inventory's `[group:vars]` sections (see `example/hosts.ini`). This is
how one codebase serves different server roles: a `backend` group enables opcache
reset and optimize steps, a `task` group enables Horizon/queue/cron restarts, and so
on. Adding a new optional step means: create `config/tasks/<name>.yml`, add a gated
`include_tasks` line to the relevant dispatcher, and document the toggle.

### Two variable namespaces

- `ansistrano_*` are consumed by the **upstream role** (`ansistrano_deploy_to`,
  `ansistrano_keep_releases`, `ansistrano_shared_paths`, `ansistrano_deploy_via`,
  and the hook-file bindings). Change these only with the upstream role's docs in mind.
- `app_*` are **this wrapper's own** convention: feature toggles plus config
  (`app_deploy_to`, `app_project_name`, `app_environment`, `app_writeable_paths`,
  `app_slack_token`, `app_current_path`, and the toggle flags). The consuming
  project supplies concrete values through its own vars file, loaded via
  `app_vars_file` (for example `build/ansible/vars.yml`).

### Dynamic deployments

`config/tasks/dynamic/` implements ephemeral per-domain deployments (spin a domain
up, tear it down). It provisions a Route53 DNS record, folder structure, storage
mounts, and an nginx vhost, all gated behind `app_dynamic_deploy`. Every provisioning
task has a teardown counterpart in `config/tasks/dynamic/cleanup/`, wired through
`cleanup.yml`. These tasks are the ones that hardcode `become: true` and delegate to
`localhost` (Route53) or target the server as root (nginx, mounts).

### Templates live in the consuming project, not here

nginx and supervisor configs are Jinja2 templates referenced as
`{{ ansistrano_deploy_from }}/build/deploy/...` (for example
`build/deploy/nginx/<env>.conf.j2`, `build/deploy/supervisor/<env>.conf.j2`). Those
`.j2` files, plus `build/.rsync_excludes` and the project vars file, are expected to
exist in the **project that installs this package**. They are intentionally absent
from this repo. Do not try to add them here.

## Conventions and gotchas

- **Release path vs current path.** Tasks that run before the symlink flip operate on
  the release being built, addressed as `{{ ansistrano_release_path.stdout }}` (see
  `optimize_pre.yml`, `set_writeable_permissions.yml`). Tasks that run after the flip
  operate on the live release, addressed as `{{ app_current_path }}` (which is
  `{{ ansistrano_deploy_to }}/current`; see `optimize_post.yml`, `restart_horizon.yml`,
  `opcache_reset.yml`). Using the wrong one silently targets the wrong release.
- **Optional privilege escalation.** Many task-level steps make `become` conditional
  via `app_become_on_*` variables defaulting to `false`, because the deploy user often
  already owns the paths. Server and dynamic tasks, by contrast, hardcode `become: true`.
- **Tags.** Two axes: lifecycle phase (`pre`, `post`) and concern (`app`, `server`,
  `dynamic`). Use them with `--tags` to run a subset.
- **`run_once` and delegation.** Cluster-wide, side-effect-once actions (Slack,
  Route53, view cache warm) use `run_once: true`, and localhost-side actions add
  `delegate_to: localhost`. Slack notifications also carry `ignore_errors: true` so a
  notification failure never fails a deploy.
- **`create_supervisor_config.yml` is deprecated** in favor of the Laravel-specific
  `config/tasks/laravel/deploy_supervisor_*_config.yml` tasks. Prefer the latter.
- **YAML indentation is 2 spaces** in this repo (`.editorconfig` sets 2 for `*.yml`,
  4 elsewhere). Match it when editing task files.
- **Never print secrets.** Slack tokens and AWS keys flow through inventory/vars at
  runtime; keep them out of committed files and command output.
