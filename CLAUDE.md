# Ansible - Ansible Configuration for Proxmox

Ansible playbooks for configuring fresh Proxmox VE installations and installing PVE on Debian.

## Quick Reference

```bash
# Post-install configuration (local via iac-driver)
cd ../iac-driver && ./run.sh --scenario pve-setup --local

# Post-install configuration (remote via iac-driver)
cd ../iac-driver && ./run.sh --scenario pve-setup --remote <IP>

# Or run playbooks directly:
ansible-playbook -i inventory/local.yml playbooks/pve-setup.yml
ansible-playbook -i inventory/local.yml playbooks/user.yml

# Install PVE on Debian 13 Trixie
ansible-playbook -i inventory/remote-dev.yml playbooks/pve-install.yml \
  -e ansible_host=<IP> -e pve_hostname=<hostname>
```

## Project Structure

```
ansible/
├── ansible.cfg           # Ansible configuration
├── collections/
│   └── ansible_collections/
│       └── homestak/
│           ├── debian/       # Debian-generic roles
│           │   ├── galaxy.yml
│           │   └── roles/
│           │       ├── base/         # System packages, timezone, locale
│           │       ├── users/        # Create local_user with sudo
│           │       ├── security/     # SSH hardening, fail2ban (prod)
│           │       └── iac_tools/    # Install packer, tofu
│           └── proxmox/      # PVE-specific roles
│               ├── galaxy.yml
│               └── roles/
│                   ├── install/      # Install PVE on Debian 13
│                   ├── configure/    # PVE config (repos, nag removal)
│                   ├── networking/   # Bridge creation, re-IP, rename, DHCP/static, IPv6
│                   └── api_token/    # Create pveum API token
├── inventory/
│   ├── local.yml         # Local execution (ansible_connection: local)
│   ├── local-dev.yml     # Local with dev group settings
│   ├── remote-dev.yml    # SSH to dev hosts (requires -e ansible_host=<IP>)
│   ├── remote-prod.yml   # SSH to prod hosts
│   └── group_vars/
│       └── all.yml       # Fallback defaults for standalone execution
├── playbooks/
│   ├── pve-setup.yml     # Core PVE config
│   ├── pve-install.yml   # Install PVE on Debian 13 Trixie
│   ├── pve-network.yml   # Network config (re-IP, rename, IPv6)
│   ├── trigger-network.yml # Push-triggers-pull for network changes
│   ├── pve-iac-setup.yml # Install IaC tools (packer, tofu)
│   └── user.yml          # User management only
└── roles/                # ansible-galaxy installed roles (gitignored)
```

## Collections

Roles are organized into two collections:

### homestak.debian

Debian-generic roles that work on any Debian host (with or without Proxmox):

| Role | Purpose |
|------|---------|
| `base` | System packages, timezone, locale |
| `users` | Create local_user with sudo (default shell: `/bin/bash` via `local_user_shell`) |
| `security` | SSH hardening, fail2ban (prod) |
| `iac_tools` | Install packer and tofu from official repos |

### homestak.proxmox

PVE-specific roles (depend on `homestak.debian`):

| Role | Purpose |
|------|---------|
| `install` | Install PVE on Debian 13 Trixie |
| `configure` | PVE config (repos, subscription nag removal) |
| `networking` | Bridge creation, re-IP, rename, DHCP/static, IPv6 |
| `api_token` | Create pveum API token for tofu |

### Role References (FQCN)

Playbooks use fully qualified collection names:

```yaml
roles:
  - homestak.debian.base
  - homestak.debian.security
  - homestak.proxmox.configure
```

## Installation

See [homestak/bootstrap](https://github.com/homestak/bootstrap) for the recommended installation method:

```bash
# One-command setup
curl -fsSL https://raw.githubusercontent.com/homestak/bootstrap/master/install | sudo bash

# Switch to homestak user, then use the 'homestak' command
sudo -iu homestak
homestak pve-setup
homestak user -e local_user=myuser
```

### Manual (without bootstrap)
```bash
git clone https://github.com/homestak-iac/ansible.git ~/iac/ansible
cd ~/iac/ansible
apt install -y ansible git
ansible-playbook -i inventory/local.yml playbooks/pve-setup.yml -c local
```

## Execution Models

### Push Model (traditional)
Remote controller SSHs to target and runs playbooks:
```bash
ansible-playbook -i inventory/remote-dev.yml playbooks/pve-network.yml \
  -e ansible_host=198.51.100.62 -e ansible_user=root ...
```
**Limitation**: SSH connection breaks when IP changes.

### Push-Triggers-Pull Model (for network changes)
Controller triggers local execution on target, avoiding SSH issues:
```bash
ansible-playbook -i inventory/remote-dev.yml playbooks/trigger-network.yml \
  -e ansible_host=198.51.100.62 \
  -e pve_network_tasks='["reip","reboot"]' \
  -e pve_new_ip=198.51.100.100 \
  -e pve_new_gateway=198.51.100.1
```
**How it works**:
1. Pushes vars to target
2. Triggers local ansible-playbook execution (async)
3. Target applies changes and reboots itself
4. Controller waits for new IP, verifies success

### Local Model (on-host)
Run directly on the PVE host:
```bash
ansible-playbook -i inventory/local.yml playbooks/pve-network.yml \
  -e pve_network_tasks='["reip","reboot"]' \
  -e pve_new_ip=198.51.100.100
```

## Configuration Source (v0.13+)

When run via iac-driver, configuration comes from config:
- `config/site.yaml` - Site defaults (timezone, packages, pve settings)
- `config/postures/{name}.yaml` - Security posture (SSH, sudo, fail2ban)
- `config/secrets.yaml` - SSH keys for authorized_keys

When run standalone, `group_vars/all.yml` provides safe fallback defaults.

**Variable precedence:** config resolved vars > extra_vars > group_vars/all.yml

## Key Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `timezone` | UTC | System timezone (config: America/Denver) |
| `packages` | minimal | Packages to install (config: full list) |
| `ssh_permit_root_login` | prohibit-password | Root SSH policy (posture-dependent) |
| `sudo_nopasswd` | false | Passwordless sudo (true for dev posture) |
| `fail2ban_enabled` | false | Enable fail2ban (true for prod posture) |
| `pve_remove_subscription_nag` | true | Remove PVE subscription popup |
| `local_user` | (none) | Non-root user to create (pass via -e) |
| `local_user_shell` | /bin/bash | Shell for local_user (users role default) |
| `ansible_host` | - | Required for remote inventories |
| `pve_hostname` | - | Required for pve-install playbook |

## Networking Role Tasks

The `homestak.proxmox.networking` role dispatches tasks based on the `pve_network_tasks` list variable:

| Task | Description |
|------|-------------|
| `bridge` | Create vmbr0 bridge from primary physical interface (auto-detects interface, address, gateway; idempotent - skips if vmbr0 exists) |
| `reip` | Change static IP address |
| `rename` | Change hostname and FQDN |
| `dhcp` | Convert interface to DHCP |
| `static` | Convert interface to static IP |
| `ipv6` | Enable/disable IPv6 |
| `reboot` | Reboot and wait for reconnection |

```bash
# Create vmbr0 on fresh PVE install
ansible-playbook -i inventory/local.yml playbooks/pve-network.yml \
  -e pve_network_tasks='["bridge"]'

# Bridge with manual overrides
ansible-playbook -i inventory/local.yml playbooks/pve-network.yml \
  -e pve_network_tasks='["bridge"]' \
  -e pve_bridge_port=eno1 -e pve_bridge_address=198.51.100.61 \
  -e pve_bridge_netmask=255.255.255.0 -e pve_bridge_gateway=198.51.100.1
```

## Boolean Variables with CLI Extra-Vars

When passing boolean variables via CLI `-e` flag, Ansible receives them as strings, not booleans. This causes conditional checks to fail unexpectedly.

**Problem:**
```bash
# CLI passes "true" as a string
ansible-playbook playbook.yml -e "my_flag=true"
```

```yaml
# This conditional may not work as expected
- name: Do something
  when: my_flag  # "true" (string) is always truthy, but "false" (string) is also truthy!
```

**Solution:** Always use the `| bool` filter for variables that might come from extra-vars:

```yaml
# Correct - handles both boolean and string inputs
- name: Do something
  when: my_flag | bool

# For negation
- name: Do something else
  when: not (my_flag | bool)
```

**When to use `| bool`:**
- Variables passed via `-e` on command line
- Variables that might be set from different sources (group_vars, extra_vars, defaults)
- Any boolean variable used in conditionals where CLI usage is possible

This pattern was discovered during v0.28.

## Related Projects

Part of the [homestak-iac](https://github.com/homestak-iac) organization:

| Repo | Purpose |
|------|---------|
| [bootstrap](https://github.com/homestak/bootstrap) | Entry point - curl\|bash setup |
| [config](https://github.com/homestak/config) | Site-specific secrets and configuration |
| [ansible](https://github.com/homestak-iac/ansible) | This project - Proxmox configuration |
| [iac-driver](https://github.com/homestak-iac/iac-driver) | Orchestration engine |
| [packer](https://github.com/homestak-iac/packer) | Custom Debian cloud images |
| [tofu](https://github.com/homestak-iac/tofu) | VM provisioning with OpenTofu |

## Playbook Details

### pve-install.yml
Installs Proxmox VE on Debian 13 Trixie following the official wiki guide. Split into two idempotent phases for local-mode reboot handling (#222):

**Phase 1 (kernel.yml):**
1. Configures hostname and /etc/hosts (removes 127.0.1.1 loopback mapping)
2. Adds Proxmox repository (no-subscription)
3. Installs Proxmox kernel and reboots

**Phase 2 (packages.yml):**
4. Installs PVE packages (proxmox-ve, postfix, open-iscsi, chrony)
5. Removes Debian kernel packages
6. Cleans up temporary repo config

The split enables idempotent re-entry after reboot — if the kernel is already installed (detected via dpkg state), phase 1 is skipped and execution continues with phase 2.

**Requirements:**
- Fresh Debian 13 (Trixie) installation
- SSH access as root
- Secure Boot disabled

### pve-setup.yml
Post-install configuration for existing PVE hosts:
- Base system packages
- SSH hardening (environment-specific)
- Proxmox repo configuration
- Bridge creation for fresh PVE installs (auto-detects primary interface, creates vmbr0)

### user.yml
Creates non-privileged sudoer user (local_user variable).

## Community Roles

### lae.proxmox (Validated)

Tested successfully on Debian 13 Trixie. Alternative to `homestak.proxmox.install`:

```bash
ansible-galaxy role install lae.proxmox
ansible-playbook -i '198.51.100.x,' playbooks/test-lae-proxmox.yml -u root
```

**Requirements:** Ansible 2.15+ (for `deb822_repository` module)

**Features:** PVE installation, clustering, storage backends, subscription nag removal, Ceph, ZFS

### DebOps (Not Recommended)

Evaluated but **not suitable** for homestak:
- Framework, not standalone roles - depends on custom plugins, 60+ global handlers
- All-or-nothing adoption required
- Overkill for homelab use case

**Conclusion:** Keep current simple roles in `homestak.debian` collection.

## GitHub Repository

- https://github.com/homestak-iac/ansible

## License

Apache 2.0 - see [LICENSE](LICENSE)
