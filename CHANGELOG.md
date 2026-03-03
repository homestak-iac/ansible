# Changelog

## Unreleased

## v0.52 - 2026-03-02

### Breaking

- **Remove dead `child-pve` role** and 4 associated playbooks (bootstrap#75)
  - `roles/child-pve/` (9 files), `playbooks/child-pve-{setup,network,repos,ssh}.yml`
  - PVE lifecycle actions in iac-driver now handle all tiered deployment tasks directly

## v0.51 - 2026-02-28

### Changed
- Rename `nested-pve` role to `child-pve` (#49)
  - Role directory: `roles/nested-pve/` → `roles/child-pve/`
  - Variables: `nested_pve_bridge_*` → `child_pve_bridge_*`
  - Playbooks: `nested-pve-{setup,network,ssh,repos}.yml` → `child-pve-{setup,network,ssh,repos}.yml`
  - API token key: `nested-pve` → `child-pve` in secrets.yaml
  - Replace infrastructure addresses with TEST-NET-2 placeholders in examples

## v0.50 - 2026-02-22

### Added
- Add bridge creation task to networking role for fresh PVE installs (#43)
  - Auto-detects primary interface, creates vmbr0, moves IP config
  - Idempotent — skips if vmbr0 already exists
  - Activated via `pve_network_tasks: ["bridge"]`
- Add `defaults/main.yml` to users role with `local_user_shell` default (iac-driver#163)
  - Prevents undefined variable error when config-apply.yml runs in cloud-init environments

### Changed
- Split PVE install into kernel and packages phases for idempotent re-entry (#222)
  - Kernel phase installs proxmox kernel and reboots
  - Packages phase installs PVE packages (skips if already done)
  - Enables local-mode install to survive reboot and resume
- Remove loopback hostname mapping from `/etc/hosts` before PVE install (#47)
  - Standard Debian maps hostname to 127.0.1.1, which PVE wiki says must be removed
- Update `ansible_default_ipv4` to `ansible_facts['default_ipv4']` for ansible-core 2.20+ (#47)

### Fixed
- Fix pve_ip defaulting to 127.0.0.1 during local PVE install (#42)
  - Use `ansible_default_ipv4.address` instead of `ansible_host` for real IP detection
  - Falls back to `ansible_host` for remote mode compatibility
- Fix nested-pve SSH key copy to include homestak user (iac-driver#133)
  - Previously only copied to root, causing SSH failures from automation_user
  - Now copies to both /root/.ssh/ and /home/homestak/.ssh/
  - Enables iac-driver to SSH as homestak to nested VMs

## v0.45 - 2026-02-02

- Release alignment with homestak v0.45

## v0.44 - 2026-02-02

- Release alignment with homestak v0.44

## v0.43 - 2026-02-01

- Release alignment with homestak v0.43

## v0.42 - 2026-01-31

- Release alignment with homestak v0.42

## v0.41 - 2026-01-31

### Fixed
- Fix nested-pve role SSH paths for non-root automation user (#34)
  - Use portable SSH paths (`~/.ssh/`) that work for both root and non-root users
  - Enables recursive scenarios with `homestak` automation user

## v0.39 - 2026-01-22

### Fixed
- Fix base role to use `packages` variable from ConfigResolver
  - Changed from `common_packages` to `packages` variable name
  - Added conditional to skip when packages list is empty
  - Aligns with site-config resolved ansible vars

## v0.28 - 2026-01-18

### Features

- Decompose nested-pve-setup.yml into granular playbooks (#49)
  - `nested-pve-network.yml`: Configure vmbr0 bridge only
  - `nested-pve-ssh.yml`: Copy SSH keys for nested VM access
  - `nested-pve-repos.yml`: Sync repos and configure PVE
  - Enables phase-level control in iac-driver scenarios
  - Original `nested-pve-setup.yml` remains for backward compatibility

### Fixed

- Add `| bool` filter to copy-files.yml conditionals for CLI extra-vars compatibility
  - Fixes "Conditional result derived from value of type 'str'" error
  - Ansible CLI `-e` passes booleans as strings, filter ensures proper handling

## v0.26 - 2026-01-17

- Release alignment with homestak v0.26

## v0.25 - 2026-01-16

- Release alignment with homestak v0.25

## v0.24 - 2026-01-16

- Release alignment with homestak v0.24

## v0.22 - 2026-01-15

### Changed

- nested-pve role now uses bootstrap installer by default (#13)
  - Production mode: Clone repos from GitHub via bootstrap/install.sh
  - Dev mode: Sync local repos via tar/unarchive (`bootstrap_use_local: true`)
  - site-config always synced separately (contains encrypted secrets)
  - New variable: `homestak_src_dir` (default: `/opt/homestak`) replaces individual `*_src_dir` variables
  - New variables: `bootstrap_branch`, `bootstrap_use_local`, `bootstrap_url`

### Fixed

- Make packer setup conditional on packer being installed (#13)
  - Bootstrap doesn't install packer by default
  - Packer init now skipped when `/opt/homestak/packer/templates` doesn't exist
- Update nested-pve role packer init for per-template directory structure

## v0.18 - 2026-01-13

- Release alignment with homestak v0.18

## v0.16 - 2026-01-11

- Release alignment with homestak v0.16

## v0.13 - 2026-01-10

### Features

- Site-config consolidation: configuration now flows from site-config
  - `group_vars/all.yml` provides fallback defaults for standalone execution
  - When run via iac-driver, resolved vars from site-config take precedence

### Changes

- Simplify `inventory/group_vars/`:
  - Remove `dev.yml`, `prod.yml`, `local.yml` (replaced by site-config postures)
  - Rewrite `all.yml` as standalone fallbacks
- Add `pve_remove_subscription_nag` variable to configure role

### Code Quality

- Fix all ansible-lint violations (209 → 0)
- Add `.ansible-lint` configuration:
  - Exclude legacy `roles/` directory
  - Skip `var-naming[no-role-prefix]` (we use collection-level prefixes)
- Enable strict lint enforcement in CI (remove `continue-on-error`)
- Add `meta/runtime.yml` to both collections
- Add `CHANGELOG.md` to both collections
- Replace `curl` with `get_url` module throughout
- Add `pipefail` to all shell tasks with pipes
- Replace `ignore_errors` with `failed_when` where appropriate

### Documentation

- Update CLAUDE.md with site-config configuration flow
- Document variable precedence order

## v0.12 - 2025-01-09

- Release alignment with homestak-dev v0.12

## v0.11 - 2026-01-08

- Release alignment with iac-driver v0.11

## v0.10 - 2026-01-08

### Documentation

- Add third-party acknowledgments for lae.proxmox role

### CI/CD

- Add GitHub Actions workflow for ansible-lint

### Housekeeping

- Enable secret scanning and Dependabot

## v0.9 - 2026-01-07

### Features

- Modular pve-install with marker file detection
  - Detects `/etc/pve-packages-preinstalled` from packer PVE image
  - Skips package installation if marker present (~17 min time savings)

### Documentation

- Update scenario name: `pve-configure` → `pve-setup`

## v0.8 - 2026-01-07

### Bug Fixes

- Fix SSH jump chain verification for nested-pve scenarios (iac-driver#21)
  - Inject outer host's public key into inner PVE's `secrets.yaml`
  - Enables `ssh -J inner_pve test_vm` from outer host
  - Key flows: outer → inner (direct), outer → inner → test (jump chain)

### Documentation

- Add SSH Key Flow section to CLAUDE.md explaining nested PVE access patterns
- Rename "E2E Testing" section to "Nested PVE Deployments" (more general)
- Update role and playbook descriptions to use "nested PVE" terminology

## v0.6 - 2026-01-06

### Collection Split (#9)

Reorganized roles into two Ansible collections for better separation of concerns:

**homestak.debian** - Debian-generic roles:
- `base` - System packages, timezone, locale
- `users` - User creation with sudo
- `security` - SSH hardening, fail2ban
- `iac_tools` - Install packer and tofu (extracted from pve-iac)

**homestak.proxmox** - PVE-specific roles:
- `install` - Install PVE on Debian 13 (from pve-install)
- `configure` - PVE config, nag removal (from proxmox)
- `networking` - Re-IP, rename, DHCP/static, IPv6 (from pve-network)
- `api_token` - Create pveum API token (extracted from pve-iac)

**Local roles** (not in collections):
- `nested-pve` - E2E testing setup (kept in roles/ for internal use)

**Breaking Changes:**
- Playbooks now use FQCN role references (e.g., `homestak.debian.base`)
- Legacy `roles/` directory deprecated
- `ansible.cfg` updated with `collections_path`

### Ansible 2.15+ Migration

- Makefile: Install Ansible via pipx (2.15+) instead of apt
- Required for `deb822_repository` module used by lae.proxmox

### Community Role Evaluation (#8)

- **lae.proxmox v1.10.0**: Successfully tested on Debian 13 Trixie
  - Installs PVE, removes subscription nag, configures repositories
  - Requires Ansible 2.15+ for `deb822_repository` module
- **DebOps**: Evaluated, not recommended (framework complexity)
- Add `playbooks/test-lae-proxmox.yml` for community role testing

### Phase 5: ConfigResolver Support

- Add iac-driver sync to nested-pve role for recursive ConfigResolver deployment
- Rename `pve-deb` to `nested-pve` in copy-files task (aligns with site-config)

### Bug Fixes

- Remove `ansible.posix.synchronize` dependency - replace with tar+unarchive pattern for ansible-core compatibility
- Fix `api_token` role idempotency check (JSON output format)

## v0.5 - 2026-01-04

Consolidated pre-release with network configuration.

### Highlights

- pve-install role for Debian 13 → PVE conversion
- pve-network role with reip, rename, dhcp, static, ipv6
- E2E tested via nested-pve-roundtrip

### Features

- Add `pve-network` role for network configuration tasks:
  - `reip` - Change static IP address
  - `rename` - Change hostname and FQDN
  - `dhcp` - Convert interface to DHCP
  - `static` - Convert interface to static IP
  - `ipv6` - Enable/disable IPv6
  - `reboot` - Reboot and wait for reconnection (handles IP changes)
- Add `pve-network.yml` playbook (push model)
- Add `trigger-network.yml` playbook (push-triggers-pull model)

### Changes

- Remove `site.yml` - combo logic moved to iac-driver `pve-configure` scenario
- Remove `install.sh`, `bootstrap.sh` - moved to [bootstrap](https://github.com/homestak-dev/bootstrap) repo
- Fix pve-install GPG key download for Debian 13 (use curl instead of get_url)
- Update docs to reference bootstrap repo and `pve-configure` scenario

## v0.1 - 2026-01-03

### Roles

- **pve-install**: Debian 13 to Proxmox VE installation
- **pve-iac**: IaC tooling (packer, tofu, API tokens)
- **nested-pve**: E2E test configuration (network, SSH keys, file deployment)
- **base**: Common system configuration
- **users**: User management
- **security**: Security hardening
- **proxmox**: Proxmox host configuration

### Infrastructure

- Branch protection enabled (PR reviews for non-admins)
- Tested via iac-driver nested-pve-roundtrip scenario
