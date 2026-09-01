maintenance_openstack_snapshot
===============================

# Purpose
Take a monthly, gate-quality OpenStack Cinder volume snapshot of every volume attached to a compute instance (or an explicit override list), and optionally prune this role's own snapshots on a retention window so manual snapshots (which never expire on their own) don't accumulate indefinitely.

# Features
- Discovers every volume attached to a target server via `openstack.cloud.server_info`'s `attached_volumes` field — no separate `openstack.cloud.volume_info` lookup needed (see "Volume discovery" below) — or snapshots an explicit `volume_ids` override list instead.
- Names every snapshot `<prefix>-<host_short>-<volume>-<YYYYMMDD>`, encoding the volume as well as the host — not just `<prefix>-<host>-<YYYYMMDD>` — so a second (or third) volume on the same host never collides with the first volume's snapshot name (see "Naming" below).
- A zero-result volume resolution (nothing attached, an empty override, or an ambiguous/no server match) fails the run loudly rather than silently reporting success with zero snapshots taken.
- Builds a per-host aggregated result fact (`maintenance_openstack_snapshot_result`: a list of `{volume, snapshot_name, changed}`) for a calling playbook to assert a fleet-wide backup gate on.
- Optional, opt-in pruning of snapshots whose name starts with `maintenance_openstack_snapshot_name_prefix` and belong to the current host, older than `maintenance_openstack_snapshot_retention_days`, with a dry-run mode on by default.
- Takes an explicit, vaulted `auth:` dict instead of assuming a `clouds.yaml` or `OS_*` environment convention that does not exist anywhere in the consuming repo today (see "Auth" below).
- All OpenStack API calls run `delegate_to: localhost` / `become: false` and `no_log: true` — the API call is always made from the controller, never the managed host, and the auth dict never leaks into logs or registered-result dumps.

# Instructions

## Required Variables

| Variable | Description |
|---|---|
| `maintenance_openstack_snapshot_auth` | Dict of OpenStack auth parameters (`auth_url`, `username`, `password`, `project_name`, and domain names if the cloud uses them) as `openstacksdk`'s password auth plugin expects. Expected to arrive already vault-encrypted from the caller. Required whenever the role does anything (backup or prune). |
| `maintenance_openstack_snapshot_server` | OpenStack server (instance) name or ID whose attached volumes get discovered and snapshotted. Required only on the backup path, and only when `maintenance_openstack_snapshot_volume_ids` is empty. |

All required variables are enforced by `ansible.builtin.assert` in `tasks/assert.ansible.yml` (for `auth`, and for `server` when discovery is actually needed) and by `tasks/backup.ansible.yml` (a zero-result volume resolution fails there too). `tasks/assert.ansible.yml` itself is gated behind `maintenance_openstack_snapshot_enable`, so a disabled role never demands credentials it won't use.

## Optional Variables

| Variable | Default | Description |
|---|---|---|
| `maintenance_openstack_snapshot_enable` | `true` | Master switch. When `false`, the role does nothing at all, including the required-variable assert. |
| `maintenance_openstack_snapshot_run_backup` | `true` | Take a snapshot this run. |
| `maintenance_openstack_snapshot_run_prune` | `false` | Prune old snapshots this run. Opt-in — deletion is never a silent side effect of taking a snapshot. |
| `maintenance_openstack_snapshot_prune_dry_run` | `true` | Report what pruning would delete without deleting it. |
| `maintenance_openstack_snapshot_volume_ids` | `[]` | Explicit override list of volume name/ID strings. When non-empty, used as-is instead of discovering `maintenance_openstack_snapshot_server`'s attached volumes. |
| `maintenance_openstack_snapshot_auth_type` | `"password"` | Which openstacksdk auth plugin `maintenance_openstack_snapshot_auth` is shaped for. Defaults to `"password"` — today's implicit behavior, kept for backward compatibility. Set to `"v3applicationcredential"` to select OpenStack Application Credential auth instead (see "Auth" below). Passed through to each module's `auth_type:` parameter — confirmed a genuine, independent module parameter on `openstack.cloud.volume_snapshot` and this collection's other modules, not nested inside `auth:`. |
| `maintenance_openstack_snapshot_region_name` | `""` | Passed through to each module's `region_name:`. Leave empty for a single-region cloud. |
| `maintenance_openstack_snapshot_name_prefix` | `"maint"` | Prefix for every object this role creates, so pruning can be scoped safely to only ever touch objects this role created. |
| `maintenance_openstack_snapshot_force` | `true` | Passed to `openstack.cloud.volume_snapshot`'s `force:`. Required `true` for this role's actual use case — see "Force" below. |
| `maintenance_openstack_snapshot_retention_days` | `14` | Snapshots older than this, whose name matches this host's naming pattern, are eligible for pruning. Must be `> 0` (enforced by an assert in `tasks/prune.ansible.yml`). |
| `maintenance_openstack_snapshot_wait` | `true` | Wait for each snapshot to reach a terminal state before moving on. |
| `maintenance_openstack_snapshot_wait_timeout` | `1800` | Seconds to wait when `maintenance_openstack_snapshot_wait` is `true`. |

### Naming

Every snapshot this role creates is named `<prefix>-<host_short>-<volume>-<YYYYMMDD>` (default prefix `maint`), e.g. `maint-mailgafr-3b7e...-20260830`:

- `<host_short>` is `inventory_hostname_short`.
- `<volume>` is the Cinder volume ID for a discovered volume (`server_info`'s `attached_volumes` only guarantees an `id` key, not a name), or whatever string (name or ID) was given in an explicit `volume_ids` override.
- `<YYYYMMDD>` comes from the controller's own clock (see "Date source" below), not the managed host's.

Encoding the volume, not just the host, is a deliberate correctness fix over the original ad hoc task this role replaces: `openstack.cloud.volume_snapshot` resolves an existing snapshot with `find_snapshot(name)` — **by name alone, ignoring which volume it belongs to** (verified directly in the installed `openstack.cloud` 2.6.0 source, `plugins/modules/volume_snapshot.py`). A host-only naming scheme (`<prefix>-<host>-<YYYYMMDD>`) would make a second volume attached to the same host resolve to the same name as the first volume's snapshot, so a rerun would report `changed: false` and silently skip snapshotting it — a real, silent backup gap on any multi-volume host. Pruning filters on this same host-scoped prefix (`<prefix>-<host_short>-`), so a prune run on one host can never touch a snapshot belonging to another host even though its own listing call is project-wide.

### Volume discovery

`tasks/backup.ansible.yml` uses `openstack.cloud.server_info`, not `openstack.cloud.volume_info` filtered by attachment. Verified directly against the installed `openstack.cloud` 2.6.0 collection (matches this role's `requirements.yml` pin): `server_info`'s own `RETURN` docs (`plugins/modules/server_info.py`) document an `attached_volumes` field on every returned server — "a list of an attached volumes. Each item in the list contains at least an `id` key" — returned unconditionally (`returned: success`, no `detailed: true` extra round-trip needed). This is confirmed in the underlying `openstacksdk` 4.13.0 resource definition (`openstack/compute/v2/server.py`): `attached_volumes` maps to the standard Nova response field `os-extended-volumes:volumes_attached`, present on the normal server GET/list response. `server_info` gives the server -> attached-volumes relationship directly in one call; `volume_info` has no equivalent "volumes attached to this server" filter of its own.

### Date source

Every task that needs "today" or "now" runs `delegate_to: localhost`. With `delegate_to`, a template's `ansible_facts` resolves against the current loop host's (`inventory_hostname`'s) own gathered facts, not the delegate's, unless `delegate_facts` is set — so `ansible_facts['date_time']` would depend on facts already gathered for the managed host in the calling play, for no actual benefit here. Instead, this role uses Jinja's `now(utc=true)` global, which the controller evaluates at template-render time regardless of `delegate_to` or fact-gathering state — it needs nothing gathered anywhere and is exactly "now" where the OpenStack API call is actually made. The retention-cutoff comparison in `tasks/prune.ansible.yml` additionally reformats `now(utc=true)` through a string round-trip into a naive datetime before subtracting, so the comparison never picks up the controller's local UTC offset the way a naive `to_datetime()`/`.timestamp()` conversion normally would.

### Auth

There is no `clouds.yaml` (`/etc/openstack/clouds.yaml`, `~/.config/openstack/clouds.yaml`, or otherwise) anywhere in the consuming repo, no `OS_*` environment variables set on the controller, and the ad hoc task this role replaces passed no `auth:`/`cloud:` parameter at all — it silently depended on ambient controller state that does not appear to exist anywhere discoverable. Rather than perpetuate that, this role takes an explicit vaulted `auth:` dict, the same way `maintenance_oci_backup` takes its OCI API key as explicit vars instead of relying on ambient credentials.

`maintenance_openstack_snapshot_auth_type` selects which openstacksdk auth plugin that `auth:` dict is shaped for, and defaults to `"password"` for backward compatibility with every caller written before this variable existed. Password auth needs `auth_url`, `username`, `password`, `project_name`/`project_id`, and `user_domain_name`/`project_domain_name` if the cloud uses domains. Setting `maintenance_openstack_snapshot_auth_type: v3applicationcredential` instead selects OpenStack Application Credential auth, which needs only `auth_url`, `application_credential_id`, and `application_credential_secret` in the `auth:` dict — no username, password, or project/domain scoping, because scoping is baked into the credential itself when it's created (OVH's Horizon: Identity → Application Credentials). A leaked application credential is individually revocable from Horizon without touching the account password, unlike password auth. `auth_type` is confirmed as a genuine, independent parameter on `openstack.cloud.volume_snapshot` (and this collection's other modules) in the installed `openstack.cloud` 2.6.0 argspec — it is not nested inside `auth:`, and not just a `clouds.yaml` concept.

### Force

`maintenance_openstack_snapshot_force` defaults `true`, not `false`, because every volume this role targets is attached to a running instance (state `in-use`) at snapshot time, and Cinder only allows snapshotting an `in-use` volume when `force` is set — an `available`-only default would make this role fail on exactly the volumes it exists to back up. The resulting snapshot is therefore crash-consistent, not application-consistent; `maintenance_borgmatic_backup` runs alongside this role in the same backup phase specifically to add an application-consistent layer (database dump hooks) on top.

## Example Playbook

```yaml
- hosts: fwdv_infra
  become: false
  tasks:
    - name: Snapshot every volume attached to this instance
      ansible.builtin.include_role:
        name: guiand888.maintenance_openstack_snapshot
      vars:
        maintenance_openstack_snapshot_run_backup: true
        maintenance_openstack_snapshot_server: "{{ inventory_hostname }}"
        maintenance_openstack_snapshot_auth: "{{ vault_openstack_auth }}"
        maintenance_openstack_snapshot_region_name: "GRA9"

- hosts: fwdv_infra
  become: false
  tasks:
    - name: Prune this role's own snapshots older than the retention window (dry run)
      ansible.builtin.include_role:
        name: guiand888.maintenance_openstack_snapshot
      vars:
        maintenance_openstack_snapshot_run_backup: false
        maintenance_openstack_snapshot_run_prune: true
        maintenance_openstack_snapshot_prune_dry_run: true
        maintenance_openstack_snapshot_auth: "{{ vault_openstack_auth }}"
        maintenance_openstack_snapshot_region_name: "GRA9"
```

Application Credential auth instead of password auth — note `auth_type` and the smaller `auth:` dict shape (no `username`/`password`/`project_name`/domain fields, since scoping is implicit in the credential):

```yaml
- hosts: fwdv_infra
  become: false
  tasks:
    - name: Snapshot every volume attached to this instance
      ansible.builtin.include_role:
        name: guiand888.maintenance_openstack_snapshot
      vars:
        maintenance_openstack_snapshot_run_backup: true
        maintenance_openstack_snapshot_server: "{{ inventory_hostname }}"
        maintenance_openstack_snapshot_auth_type: v3applicationcredential
        maintenance_openstack_snapshot_auth:
          auth_url: "{{ vault_openstack_auth_url }}"
          application_credential_id: "{{ vault_openstack_app_cred_id }}"
          application_credential_secret: "{{ vault_openstack_app_cred_secret }}"
        maintenance_openstack_snapshot_region_name: "GRA9"
```

# Requirements
- `openstack.cloud` (Ansible Galaxy collection, `requirements.yml` pinned `>=2.6.0`) for `server_info`, `volume_snapshot`, and `volume_snapshot_info`.
- The `openstacksdk` Python package importable by the controller's Ansible interpreter (the collection's own requirement; not a role dependency to install separately).

Not a role dependency (`meta/main.yml` `dependencies: []` is correct) — install with `ansible-galaxy collection install -r requirements.yml`.

# Compatibility
Targets RHEL-family (EL 8/9, Fedora) and Debian-family (Debian bullseye/bookworm, Ubuntu focal/jammy/noble) managed hosts, though every OpenStack API call in this role runs `delegate_to: localhost` and never touches the managed host's OS directly — the platform list matters only in that it constrains where `inventory_hostname_short` is evaluated. Requires Ansible >= 2.12 and the `openstack.cloud` collection (`requirements.yml`, pinned `>=2.6.0`) plus the `openstacksdk` Python package importable by the controller's Ansible interpreter. Module behavior cited in this README and in task comments was verified directly against the `openstack.cloud` 2.6.0 collection and `openstacksdk` 4.13.0 sources installed in this development environment, which match the pinned requirement. Syntax-check, ansible-lint, and yamllint were run on a Fedora development host with `maintenance_openstack_snapshot_enable: false`; not all listed platforms were exercised directly, and no test in this repo makes a real OpenStack API call.

# License
- AGPLv3

# Maintainers
Guillaume A.
  - Contact: [mail@guillaumea.fr](mailto:mail@guillaumea.fr)
  - Blog: https://blog.guillaumea.fr
