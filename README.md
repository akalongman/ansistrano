# Ansistrano Wrapper

A configuration skeleton around the community [`ansistrano.deploy`](https://ansistrano.com/)
and `ansistrano.rollback` Ansible Galaxy roles. Those roles provide Capistrano style,
zero-downtime, release-based deployment: each deploy lands in its own
`releases/<timestamp>/` directory, then a `current` symlink is atomically flipped to
point at it. Rolling back is just moving that symlink to the previous release.

This wrapper adds no deployment engine of its own. Its job is to inject PHP and Laravel
specific steps (cache warming, opcache reset, php-fpm reload, queue, Horizon, Reverb,
Supervisor, cron, Slack notifications, and more) into Ansistrano's lifecycle hooks, each
one toggled per server group by a simple variable.

## Requirements

* Ansible on the control machine (the machine that runs the deploy).
* The two Galaxy roles listed in `requirements.yml` (`ansistrano.deploy` 3.14.0,
  `ansistrano.rollback` 3.1.0).
* SSH access from the control machine to your target servers.

## Installation

Install this package into your PHP project as a dev dependency, then install the Galaxy roles.

```bash
composer require --dev longman/ansistrano
ansible-galaxy install -r vendor/longman/ansistrano/requirements.yml
```

The package expects to live at `vendor/longman/ansistrano/` inside your project. That
placement matters: `vars/defaults.yml` sets `ansistrano_deploy_from` to
`{{ playbook_dir }}/../../..`, which walks three levels up from the package to your
project root. The project root is what gets rsynced to the servers.

## What your project must provide

The playbooks reference files that live in the consuming project, not in this package.
Create these before your first deploy.

| Path (relative to your project root) | Purpose |
|---|---|
| `build/ansible/vars.yml` | Your project's own Ansible variables, loaded via `app_vars_file`. |
| `build/ansible/hosts.ini` | Your inventory (copy and adapt `example/hosts.ini`). |
| `build/.rsync_excludes` | Paths excluded from the rsync (referenced through `ansistrano_rsync_extra_params`). |
| `build/deploy/nginx/<env>.conf.j2` | nginx vhost template, used only by dynamic deployments. |
| `build/deploy/supervisor/<env>.conf.j2` | Supervisor program template (Horizon and Reverb variants use `<env>-horizon.conf.j2` and `<env>-reverb.conf.j2`). |

See `example/vars.yml` and `example/hosts.ini` for starting points.

## Configuration

Configuration flows through two variable namespaces.

* `ansistrano_*` variables are consumed by the upstream role. They control the release
  mechanics (`ansistrano_deploy_to`, `ansistrano_keep_releases`, `ansistrano_shared_paths`,
  `ansistrano_deploy_via`, and the hook file bindings). Sensible defaults live in
  `vars/defaults.yml`.
* `app_*` variables are this wrapper's own convention. They fall into two groups: core
  settings that every deploy needs, and boolean feature toggles that turn individual
  post-deploy steps on or off.

### Core variables

Set these in your inventory (see `example/hosts.ini`) or in `build/ansible/vars.yml`.

| Variable | Meaning |
|---|---|
| `app_deploy_to` | Absolute target path on the server (for example `/var/www/project.com`). Maps to `ansistrano_deploy_to`. |
| `app_project_name` | Human readable project name, used in task and notification titles. |
| `app_environment` | Environment name (`staging`, `production`, ...). Selects template variants. |
| `app_vars_file` | Path to your project vars file, relative to project root (for example `build/ansible/vars.yml`). |
| `app_writeable_paths` | List of paths made writeable when `app_set_writeable_permissions` is on. |
| `app_deployer_user`, `app_deployer_group` | Ownership applied to created files and directories. Default to `ansible_user`. |
| `app_slack_token`, `app_slack_channel` | Slack credentials. A defined, non-empty token enables notifications. |

### Feature toggles

Each optional step is gated by a variable. Boolean toggles use
`app_foo|bool == true`; a few are presence based, meaning the step runs when the variable
is simply defined. Set toggles per server group in your inventory's `[group:vars]`
sections so each role (web, backend, task) runs only what it needs.

Steps run before the `current` symlink is flipped (they operate on the new release being
built):

| Toggle | Type | What it does |
|---|---|---|
| `app_sync_storage_folder` | bool | Syncs the local `storage/` into the server's `shared/` before code update. |
| `app_set_writeable_permissions` | bool | `chmod -R 777` on each path in `app_writeable_paths`. Escalates only if `app_become_on_set_permissions` is true. |
| `app_symlink_storage_public` | bool | Symlinks `storage/app/public` to `public/storage` in the new release. |
| `app_doctrine_prepare` | bool | Generates Doctrine proxies and clears the metadata cache. |
| `app_generate_passport_keys` | bool | Runs `passport:keys` (errors ignored). |
| `app_optimize_pre` | bool | Warms `config:cache`, `route:cache`, and `event:cache`. Override the routes command with `app_cmd_cache_routes`. |
| `app_run_self_diagnosis` | bool | Runs `php artisan self-diagnosis <env>`. |
| `app_doctrine_update` | bool | Runs `doctrine:schema:update --force`. |

Steps run after the symlink flip (they operate on the live `current` release):

| Toggle | Type | What it does |
|---|---|---|
| `app_restart_pm2` | bool | Restarts a PM2 process named by `app_pm2_script_name`. |
| `app_reset_opcache` | bool | Resets opcache via `cachetool`. Variant is chosen by presence of `app_fcgi_connection` or `app_cachetool_config`, otherwise CLI. |
| `app_restart_phpfpm` | bool | Reloads php-fpm through systemctl. |
| `app_optimize_post` | bool | Warms `view:cache`. |
| `app_deploy_supervisor_horizon_config` | bool | Renders and installs the Horizon Supervisor config. |
| `app_deploy_supervisor_reverb_config` | bool | Renders and installs the Reverb Supervisor config. |
| `app_restart_supervisor` | bool | Restarts the Supervisor service (name via `app_supervisor_service_name`, default `supervisord`). |
| `app_restart_queue` | bool | Runs `queue:restart`. |
| `app_restart_horizon` | bool | Runs `horizon:terminate` so Horizon respawns on the new release. |
| `app_add_cron_entry` | bool | Installs a per-minute `schedule:run` cron entry. |

The presence of a non-empty `app_slack_token` enables a success notification at the end
of a deploy and a failure notification if any task fails.

## Usage

Run the playbooks from your project root, pointing `-i` at your inventory and naming the
playbook inside the package.

```bash
# Deploy
ansible-playbook -i build/ansible/hosts.ini vendor/longman/ansistrano/deploy.yml

# Roll back one release
ansible-playbook -i build/ansible/hosts.ini vendor/longman/ansistrano/rollback.yml

# Tear down a dynamic deployment
ansible-playbook -i build/ansible/hosts.ini vendor/longman/ansistrano/cleanup.yml
```

`deploy.yml` and `cleanup.yml` set `any_errors_fatal: true`, so a failure on any host
aborts the whole run.

Useful selectors while iterating:

```bash
ansible-playbook ... deploy.yml --syntax-check                      # parse only
ansible-playbook ... deploy.yml --check                            # dry run
ansible-playbook ... deploy.yml --limit task                       # one inventory group
ansible-playbook ... deploy.yml --tags pre                         # one lifecycle phase
ansible-playbook ... deploy.yml --start-at-task "APP | Restart Horizon"
```

Tasks are tagged on two axes: lifecycle phase (`pre`, `post`) and concern (`app`,
`server`, `dynamic`). Combine with `--tags` to run a subset.

## Deployment lifecycle

Ansistrano exposes hook points around its run. This wrapper binds a dispatcher file to
each one in `vars/defaults.yml`. Every dispatcher is a list of conditional includes, so a
step only runs when its toggle is set.

| Hook (`config/`) | When it runs | Typical work |
|---|---|---|
| `before_setup.yml` | before the release directory exists | Dynamic infrastructure (DNS, folders, mounts, nginx). |
| `before_update_code.yml` | before code is synced | Sync storage content. |
| `before_symlink.yml` | new release built, not yet live | Permissions, cache warm, self-diagnosis, Doctrine. |
| `after_symlink.yml` | new release is now live | opcache reset, service restarts, cron, Slack. |
| `after_cleanup.yml` | after old releases are pruned | Copyright notice. |
| `on_failure.yml` | any task fails | Slack failure notification. |

## Dynamic deployments

Setting `app_dynamic_deploy` to true enables an ephemeral, per-domain deployment mode. On
deploy it provisions a Route53 DNS record, the folder structure, NFS storage mounts, and
an nginx vhost for `app_domain`. Each provisioning step has a teardown counterpart, all
wired through `cleanup.yml`, so running the cleanup playbook removes everything the deploy
created. The nginx vhost is further gated by `app_dynamic_create_vhost`, and the storage
mount by the presence of `app_dynamic_mount_path`. These tasks run with privilege
escalation and require AWS credentials (`app_aws_access_key_id`, `app_aws_secret_access_key`)
for the DNS step.

## Notes

* Two task files ship but are not wired into any hook: `composer_optimize.yml` and
  `symlink_uploads.yml`. Include them yourself if you need them.
* `create_supervisor_config.yml` and `restart_supervisor_instance.yml` are deprecated.
  Prefer the Laravel specific Supervisor tasks under `config/tasks/laravel/`.
* `restart_phpfpm.yml` reloads a hardcoded `php74-php-fpm` unit. Adjust it for your PHP
  version.
* The automatic rollback on failure is present but commented out in `on_failure.yml`.

## License

Released under the MIT License. See `LICENSE`. Maintained by Avtandil Kikabidze
(akalongman@gmail.com).
