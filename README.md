# ansible-role-tpotce

An Ansible **role** implementation of the [T-Pot](https://github.com/telekom-security/tpotce)
installer. Upstream ships a monolithic playbook (`installer/install/tpot.yml`) driven by
`install.sh`; this role does the same work, split into logical task files, so it can be
consumed like any other role.

The role builds **sensor-only** deployments: honeypots plus `tpotinit`, with logs shipped off
the box by Filebeat. It deliberately never deploys the web UI or the local
Elasticsearch / Logstash / Kibana stack.

Tracks upstream T-Pot **24.04.1**.

## Differences from upstream

| | Upstream | This role |
|---|---|---|
| Refuses to run as `root` | yes | **no**, on purpose — the role can be driven by provisioning systems that run as root |
| `docker-compose.yml` | `install.sh` copies a prebuilt edition, or you run `compose/customizer.py` interactively | generated from `tpot_services` (see below) |
| Elasticsearch / Kibana / Logstash / nginx / spiderfoot / AttackMap | available | never deployed |
| Filebeat | not included | optional container, shipping to your own Logstash |
| Web UI user (`WEB_USER`) | prompted for | not applicable, sensors have no web UI |
| Clone location | `/home/<user>/tpotce` | `/opt/tpotce`, so it does not depend on which account ran the role |

## Requirements

- One of: AlmaLinux, Debian, Fedora, openSUSE Tumbleweed, Raspbian, RHEL, Rocky, Ubuntu
- The `ansible.posix` collection (`ansible-galaxy collection install -r requirements.yml`)

## Example playbook

```yaml
- hosts: sensors
  become: true
  roles:
    - role: ansible-role-tpotce
      vars:
        tpot_user: tsec
        tpot_services:
          - cowrie
          - dionaea
          - heralding
          - suricata
          - filebeat
```

After the run, reboot and reconnect on **tcp/64295** — the role moves SSH off port 22 so the
honeypots can use it.

## How `docker-compose.yml` is generated

Upstream's `compose/customizer.py` is an interactive script: it reads
`compose/tpot_services.yml` (the catalogue of every service T-Pot knows about), asks
`Include <service>? (y/n)` for each one, enforces a few dependency rules, drops unused
networks and warns about port conflicts.

This role reproduces that logic in `tasks/11_build_compose.yml`, with `tpot_services`
supplying the answers instead of a human. The catalogue is read **from the cloned repository
at run time**, so the role carries no copy of upstream's compose YAML and picks up new
honeypots automatically.

What is reproduced from `customizer.py`:

- `tpotinit` is always included
- selecting any Snare / Tanner service pulls in the whole group
- `honeytrap` and `glutton` cannot both run; `honeytrap` wins
- networks not referenced by a selected service are dropped
- host port conflicts are detected

Two deliberate differences:

- **Port conflicts are a hard failure.** `customizer.py` only warns, because a human reviews
  the file afterwards. A role has no such reviewer and a conflicting file fails at
  `docker compose up`, so the run stops with the conflicting services named.
- **`kibana → elasticsearch`, `spiderfoot → nginx` and `map_* → elasticsearch + nginx` are not
  implemented**, because none of those services can ever be selected here.

### Port de-confliction

`tpot_services.yml` lists every port a honeypot is *capable* of binding — `heralding` alone
claims 21, 22, 23, 25, 80, 443, 3306 and 3389 — so a naive selection collides immediately.
Upstream resolves this in its prebuilt editions by commenting out the losing side, and those
editions are otherwise identical to the catalogue.

The role therefore takes the port list for any selected service that also appears in
`tpot_port_reference_edition` (default `sensor`) from that edition, inheriting upstream's own
conflict resolution rather than hardcoding a port table that would drift. Anything left over is
reported by the conflict check, and can be resolved with `tpot_port_overrides`.

## Role variables

### Selection

| Variable | Default | Description |
|---|---|---|
| `tpot_services` | upstream's `sensor` list, minus `logstash` | Services to deploy. `tpotinit` is implicit. |
| `tpot_port_reference_edition` | `sensor` | Compose edition to inherit port lists from. `''` disables. |
| `tpot_port_overrides` | `{}` | Per-service port lists, applied last. |

Selectable services are whatever upstream's catalogue defines, plus `filebeat`. Requesting an
unknown or excluded service fails the run with the list of valid names.

### Paths and identity

| Variable | Default | Description |
|---|---|---|
| `tpot_user` | `{{ ansible_facts['user_id'] }}` | User T-Pot is installed for; added to `docker` and `tpot`. |
| `tpot_user_home` | `{{ ansible_facts['user_dir'] }}` | Home directory of that user. |
| `tpot_dir` | `/opt/tpotce` | Clone location. Upstream uses the invoking user's home directory; `/opt` keeps the path identical on every host. Created with `become`, owned by `tpot_user`. |
| `tpot_data_path` | `{{ tpot_dir }}/data` | Host side of `TPOT_DATA_PATH`; keep in sync if you override it. |
| `tpot_repo` / `tpot_repo_version` | upstream / `master` | Repository to clone. |

### Configuration

| Variable | Default | Description |
|---|---|---|
| `tpot_env` | `TPOT_TYPE: SENSOR`, empty `WEB_USER`/`LS_WEB_USER` | Keys written into `.env`. Everything else keeps upstream's shipped value. |
| `tpot_env_extra` | `{}` | Additional `.env` keys, merged over `tpot_env`. |
| `tpot_pull_images` | `true` | Run `docker compose pull` after generating the compose file. |
| `tpot_daily_reboot` | `true` | Install upstream's nightly prune-and-reboot cron job. |
| `tpot_daily_reboot_hour` / `_minute` | seeded from `inventory_hostname` | Schedule. Seeded rather than re-randomised so a sensor keeps its slot. |

Example — join a hive and change the data path:

```yaml
tpot_env_extra:
  TPOT_HIVE_IP: 10.0.0.5
  TPOT_DATA_PATH: /srv/tpot/data
tpot_data_path: /srv/tpot/data
```

### Filebeat

Add `filebeat` to `tpot_services` to ship logs to your own Logstash. The container mounts the
configuration written by
[honeynet/ansible-role-tpotce-filebeat](https://github.com/honeynet/ansible-role-tpotce-filebeat),
which you run alongside this role.

| Variable | Default |
|---|---|
| `filebeat_image` | `quay.io/honeynet/filebeat` |
| `filebeat_version` | `7.11.1` |
| `filebeat_path` | `/opt/filebeat/etc` |

Filebeat reads `MY_EXTIP` / `MY_INTIP` / `MY_HOSTNAME` from
`${TPOT_DATA_PATH}/tpot/etc/compose/elk_environment`, which `tpotinit` writes at start up.
Because Docker Compose resolves `env_file` while *parsing* the configuration, the role creates
a placeholder first — without it the very first `docker compose up` aborts.

> **Note:** the companion role's `conf.d` files were written for T-Pot 19.x/20.06 and do not yet
> cover honeypots added since (`beelzebub`, `ddospot`, `galah`, `h0neytr4p`, `honeyaml`,
> `miniprint`, `sentrypeer`, `wordpot`, …), while some it does cover (`honeypy`, `honeysap`,
> `rdpy`) no longer exist upstream.

## Package lists

Packages are not listed in the tasks. Each supported platform has a file under `vars/`
(`Debian.yml`, `Fedora.yml`, `AlmaLinux.yml`, `Rocky.yml`, `RedHat.yml`, `Suse.yml`), selected
by `ansible_distribution` and falling back to `ansible_os_family`. The tasks only reference
`tpot_recommended_packages`, `tpot_conflicting_packages`, `tpot_docker_packages`,
`tpot_docker_packages_remove`, `tpot_docker_repo_url`, `tpot_grc_rpm_url` and
`tpot_cron_package`.

`tpot_recommended_packages` is a faithful mirror of upstream's list for each
distribution, so it can be diffed against the installer. What actually gets
installed is `tpot_packages`, combined at run time as
`(tpot_recommended_packages + [tpot_cron_package]) | unique`. The cron package
is kept separate because upstream schedules a daily reboot on all eight
distributions but only installs a cron implementation on the Debian family and
Fedora — on AlmaLinux, RHEL, Rocky and openSUSE its job has nothing to run it.
On the distributions upstream already covers, the merge is a no-op.

## Task layout

Task files run in the order upstream's plays do — note that the group must exist before the
user is added to it, and the repository must be cloned before anything reads from it.

| File | Purpose |
|---|---|
| `01_bootstrap_python.yml` | Install python3 via `raw` |
| `02_support_check.yml` | Refuse unsupported distributions and the reserved `tpot` user |
| `03_prep_system.yml` | Recommended / conflicting packages, grc, micro |
| `04_prep_for_docker.yml` | Remove distro Docker packages, add Docker repository |
| `05_install_docker.yml` | Install Docker Engine |
| `06_setup_system.yml` | `tpot` user and group, sysctl, sshd, firewalld, SELinux, resolved |
| `07_restart_services.yml` | Restart resolved, firewalld, Docker, SSH |
| `08_config_system.yml` | Shell aliases, clone the repository, group membership |
| `09_setup_tpot.yml` | Install and enable `tpot.service` |
| `10_config_tpot.yml` | Write `.env`, seed the filebeat `elk_environment` placeholder |
| `11_build_compose.yml` | Generate `docker-compose.yml` (the customizer replacement) |
| `12_pull_images.yml` | `docker compose pull` |
| `13_daily_reboot.yml` | Randomized nightly reboot cron job |

## Testing

```bash
pip install -r requirements.txt
ansible-galaxy collection install -r requirements.yml

yamllint . && ansible-lint     # lint
molecule test                  # default scenario, in Docker
molecule test -s full          # full install, needs Vagrant + libvirt
```

The `default` scenario covers `.env` handling and compose generation — it asserts the Snare /
Tanner group expands, glutton loses to honeytrap, excluded services never appear, networks are
pruned, no two services share a host port, and the filebeat paths are correct. It does not
install Docker Engine or start honeypots; the `full` scenario does that on a throwaway VM.

## Special Thanks

<p>This project is supported by:</p>
<p>
  <a href="https://www.digitalocean.com/">
    <img src="https://opensource.nyc3.cdn.digitaloceanspaces.com/attribution/assets/SVG/DO_Logo_horizontal_blue.svg" width="201px">
  </a>
</p>

## License

The role is licensed under GPLv3
