# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Hard constraints

- **Never run this role or an equivalent playbook against the local machine.** It rewrites
  `sshd_config` (moving SSH to tcp/64295), installs a systemd unit, replaces the distribution's
  Docker packages and installs a nightly reboot cron job. Running it locally breaks the machine.
  Testing happens in Molecule containers or a throwaway VM only.
- **The role deliberately does not check whether it is running as `root`.** Upstream's playbook
  refuses to run as root; that check was removed on purpose so provisioning systems can drive the
  role. Do not reinstate it.
- Task files are split by concern on purpose. Keep the split; do not consolidate them back into a
  single `main.yml`.

## What this repository is

An Ansible **role** port of the T-Pot installer from
[telekom-security/tpotce](https://github.com/telekom-security/tpotce) (tracking **24.04.1**),
whose installer is a monolithic playbook (`installer/install/tpot.yml`) driven by `install.sh`.

The role builds **sensor-only** deployments: honeypots plus `tpotinit`, with logs shipped off the
box by Filebeat. It never deploys the web UI or a local ELK stack.

Upstream is the reference for behaviour. When changing a task, check what the corresponding
upstream play does first — the task names in `tasks/*.yml` mirror upstream's play names
(including their `(All)` / `(Debian)` suffixes) precisely so the two can be diffed.

## Common commands

```bash
pip install -r requirements.txt
ansible-galaxy collection install -r requirements.yml   # ansible-core ships no collections

yamllint . && ansible-lint            # both must pass; CI runs exactly these

molecule test                         # default scenario (Docker), debian13
molecule converge && molecule verify  # iterate without tearing the container down
molecule destroy                      # clean up afterwards

# One distro from the CI matrix. All three vars must be set together; the
# image is a full reference, not a distro slug.
MOLECULE_IMAGE=almalinux:10 MOLECULE_DISTRO_TAG=almalinux10 \
  MOLECULE_DOCKER_COMMAND=/usr/lib/systemd/systemd molecule test

molecule test -s full                 # full install; needs Vagrant + libvirt, VM only
```

Run only one Molecule scenario at a time, or set `MOLECULE_EPHEMERAL_DIRECTORY` per invocation —
parallel runs share the ephemeral directory and corrupt each other's inventory.

## Architecture

### Execution order is load-bearing

`tasks/main.yml` includes `01_bootstrap_python.yml` … `14_set_hostname.yml` in the order
upstream's plays run. The numbering is not cosmetic:

- `06_setup_system.yml` creates the `tpot` group; `08_config_system.yml` adds the user to it.
- `08_config_system.yml` clones the repository into `tpot_dir`; everything after it reads from
  that clone — including `11_build_compose.yml`, which reads upstream's service catalogue from
  the working tree at run time.

Upstream's own playbook has these two steps in the wrong order (a defect this role fixes), so
"matches upstream" is not by itself a reason to reorder them back.

### Variable layering

Three layers, highest precedence last:

1. `defaults/main.yml` — everything a consumer is meant to override (`tpot_services`, `tpot_dir`,
   `tpot_user`, `tpot_env`, filebeat settings, feature toggles).
2. `vars/main.yml` — role constants: the excluded-service list, dependency groups, the filebeat
   container definition, and *fallbacks* for every package variable.
3. `vars/<distribution>.yml`, loaded by `tasks/main.yml` via `first_found` on
   `ansible_distribution` then `ansible_os_family` (`Debian.yml`, `Fedora.yml`, `AlmaLinux.yml`,
   `Rocky.yml`, `RedHat.yml`, `Suse.yml`).

**No task may contain a literal package name.** Tasks reference `tpot_packages`,
`tpot_conflicting_packages`, `tpot_docker_packages`, `tpot_docker_packages_remove`,
`tpot_docker_repo_url`, `tpot_grc_rpm_url`, `tpot_cron_package` only.

EPEL is the role's own responsibility, not upstream's: `htop` is not in the AlmaLinux, Rocky or
RHEL base repositories, and upstream enables EPEL in `install.sh` rather than in the playbook. A
role that replaces both has to do it, via `tpot_epel_package` / `tpot_epel_rpm_url`. Note that the
`geerlingguy` Rocky image ships EPEL already, so this gap is invisible on that platform and only
shows up on the stock `almalinux:10` image.

`tpot_recommended_packages` is kept byte-identical to upstream's list per distribution so it can
be diffed against the installer. What is actually installed is `tpot_packages`, defined in
`vars/main.yml` as `(tpot_recommended_packages + [tpot_cron_package]) | unique` — this works
because Ansible templates variables lazily, so it resolves *after* the distribution file has
loaded. The cron package is separate because upstream schedules a daily reboot on all eight
distributions but only installs cron on the Debian family and Fedora.

### Compose generation replaces upstream's `customizer.py`

`tasks/11_build_compose.yml` + `templates/tpot/tpot.yml.j2` reproduce
`compose/customizer.py`, with `tpot_services` supplying the answers instead of a human. The
catalogue (`compose/tpot_services.yml`) is **slurped from the remote clone** — the role carries
no copy of upstream's compose YAML, so new honeypots appear automatically. Use `slurp`, never
`lookup('file')`, which would read from the controller.

Reproduced from `customizer.py`: `tpotinit` always included; any Snare/Tanner service pulls in
the whole group; `honeytrap` beats `glutton`; unused networks pruned; port conflicts detected.

Two deliberate departures:

- **Port conflicts are a hard failure**, not a warning — a role has no human reviewer, and a
  conflicting file dies at `docker compose up`.
- The `kibana → elasticsearch`, `spiderfoot → nginx` and `map_* → elasticsearch + nginx` rules
  are unimplemented because those services are permanently excluded.

`tpot_excluded_services` (elasticsearch, kibana, logstash, map_data, map_redis, map_web, nginx,
spiderfoot) can **never** be selected. This is a product decision, not an oversight. `filebeat`
is a role-local addition defined in `vars/main.yml` (`tpot_local_services`), not an upstream
service.

**Port de-confliction:** the catalogue lists every port a honeypot is *capable* of binding
(`heralding` alone claims 21, 22, 23, 25, 80, 443, 3306, 3389), so a naive selection collides.
Ports for any selected service that also appears in `tpot_port_reference_edition` (default
`sensor`) are taken from that prebuilt edition, inheriting upstream's own conflict resolution
instead of hardcoding a table that would drift. `tpot_port_overrides` patches the remainder.

The template emits **no generation timestamp**, deliberately, so re-running against an unchanged
configuration leaves `docker-compose.yml` untouched.

### Kernels without xtables

RHEL 10 and its rebuilds ship a kernel with no legacy xtables modules, which breaks two things
`06_setup_system.yml` handles by probing rather than assuming: `modprobe iptable_filter` is fatal
there (so it is gated behind a `modprobe -n` dry run), and Docker's iptables firewall backend
cannot load `xt_addrtype` (so `/etc/docker/daemon.json` is merged — never overwritten — with
`{"firewall-backend": "nftables"}` and Docker is restarted through a handler).

The same file also sets `net.ipv4.ip_forward=1`, persisted and live. Docker used to enable
forwarding itself; with the nftables backend it refuses to start instead.

### Handlers

Firewalld, SSH and the `tpot.service` unit are reloaded through `handlers/main.yml`, not by
inline restart tasks as upstream does. `07_restart_services.yml` opens with
`meta: flush_handlers`, and `tasks/main.yml` closes with a second one so the unit handler
notified by `09_setup_tpot.yml` runs before the role returns rather than at the end of the
play, after whatever roles follow. systemd-resolved and Docker stay as ordinary tasks there — see the
comments in that file for why.

### `.env` and the filebeat placeholder

`10_config_tpot.yml` seeds `{{ tpot_dir }}/.env` from upstream's `env.example` and then rewrites
only the keys in `tpot_env` / `tpot_env_extra`; every other key keeps upstream's shipped value.

It also creates a placeholder at `${TPOT_DATA_PATH}/tpot/etc/compose/elk_environment`. Docker
Compose resolves `env_file` at **parse** time, so without that file the very first
`docker compose up` aborts before `tpotinit` can write the real one.

## Testing notes

The `default` scenario runs **the whole role** (`converge.yml` uses `roles:`), never individual
task files. `tpot_pull_images: false` there — pulling honeypot images would fetch tens of
gigabytes per CI run.

- **No idempotence step.** The role mirrors upstream, which probes with `raw` and unconditionally
  restarts services, so a second converge legitimately reports changes.
- `ansible_become: false` is set as a host var. The container connection is already root, and the
  `rockylinux9` image has no sudo password (`PAM account management error`). As a connection
  variable it overrides the play's `become: true`, which `converge.yml` keeps so the role is
  exercised the way a consumer runs it. `prepare.yml` likewise has no `become`.
- **Every platform is built from `molecule/default/Dockerfile.j2`** (`pre_build_image: false`),
  because the matrix is not uniform. geerlingguy publishes an Ansible-ready image for Debian,
  Ubuntu, Rocky and Fedora; AlmaLinux and openSUSE Tumbleweed have none, so those start from the
  distribution's own image and the Dockerfile adds systemd, python3 and sudo. systemd must be
  baked into the image — it is PID 1, so `prepare.yml` cannot install it. For the geerlingguy
  bases the build layer is effectively a no-op.
- `prepare.yml` then installs what a real host ships but the images strip (openssh-server,
  systemd-resolved, kmod, firewalld) and swaps `curl-minimal` for `curl` with `allowerasing` on
  the RedHat family — a container artifact, not a role problem.
- `/lib/modules` is mounted read-only so the `modprobe` probes in `06_setup_system.yml` can
  resolve module names. Those probes ask the **host** kernel, not the container's distribution,
  so a runner with full xtables makes every distro take the xtables-present path — the nftables
  fallback is not reachable from container CI and belongs to the `full` scenario on an EL10 VM.
- CI matrix: debian13, ubuntu2604, almalinux10, rockylinux10, fedora44, tumbleweed. **It carries
  the current release of each distribution and nothing older** — upstream's installer asserts the
  distribution version and refuses anything else, so an older release is not a supported
  configuration and proves nothing. When upstream bumps its version map, *move* the matrix rather
  than extending it. RHEL and Raspbian have no usable public image and rely on the `full` scenario.
- Keep the run warning-free. Use `ansible_facts['x']` rather than bare injected facts
  (`INJECT_FACTS_AS_VARS` is removed in ansible-core 2.24) and `deb822_repository` rather than
  `apt_repository` (removed in 2.25). `deb822_repository` needs `python3-debian`, which is not
  universally present, hence `install_python_debian: true` in `04_prep_for_docker.yml`.

### Verifying locally

The sandbox cannot reach `mirrors.almalinux.org` or `download.opensuse.org` (403 from the network
policy), and it is aarch64 while the runners are x86_64 — so the `almalinux10` and `tumbleweed`
jobs cannot be reproduced here and CI is the only check for them. Its kernel also has no
netfilter modules, so firewalld will not start: every EL and Tumbleweed scenario fails at
`Get Firewall rules` locally, on the unmodified role too. Debian and Ubuntu run clean and are the
fast local signal.

`.ansible-lint` skips `package-latest`, `no-changed-when`, `command-instead-of-module` and
`name[casing]` — all four because the role tracks upstream's behaviour and naming. Prefer fixing
a task over widening that list.

## Branches

`full-rework` is the development branch and CI's push target; `master` is the release branch.
Work is submitted as **stacked PRs** grouped by logical change, each based on the previous.
