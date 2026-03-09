# ansible

Ansible playbooks for Proxmox VE configuration and PVE installation.

Part of the [homestak-iac](https://github.com/homestak-iac) organization. See [bootstrap](https://github.com/homestak/bootstrap) for installation.

## Playbooks

| Playbook | Description |
|----------|-------------|
| `pve-setup.yml` | Core PVE config (packages, SSH, subscription nag) |
| `user.yml` | User management (create user, sudo, SSH keys) |
| `pve-install.yml` | Install PVE on Debian 13 Trixie |

## Install PVE on Debian

Install Proxmox VE on a fresh Debian 13 (Trixie) system:

```bash
ansible-playbook -i inventory/remote-dev.yml playbooks/pve-install.yml \
  -e ansible_host=198.51.100.100 \
  -e pve_hostname=pve-new
```

Requirements:
- Fresh Debian 13 Trixie with SSH access
- Static IP configured
- Secure Boot disabled

## Inventory Options

| Inventory | Connection | Use case |
|-----------|------------|----------|
| `local.yml` | local | Running on the host you're configuring |
| `local-dev.yml` | local | Local with dev tools (strace, tcpdump) |
| `remote-dev.yml` | SSH | Remote dev/test hosts |
| `remote-prod.yml` | SSH | Remote production (strict SSH, fail2ban) |

## What It Does

- Creates non-privileged user with sudo
- Disables enterprise repos, enables no-subscription repo
- Installs common packages (htop, git, vim, tmux, etc.)
- Configures SSH hardening
- Removes subscription nag from web UI

## Structure

```
collections/
└── ansible_collections/
    └── homestak/
        ├── debian/           # Debian-generic roles
        │   └── roles/
        │       ├── base/         # Packages, timezone
        │       ├── users/        # User creation, sudo, SSH keys
        │       ├── security/     # SSH hardening, fail2ban
        │       └── iac_tools/    # Install packer, tofu
        └── proxmox/          # PVE-specific roles
            └── roles/
                ├── install/      # Install PVE on Debian
                ├── configure/    # Subscription nag, repos
                ├── networking/   # Bridge creation, re-IP, rename, DHCP/static
                └── api_token/    # pveum API token
roles/
└── nested-pve/           # Integration testing (not in collections)
playbooks/
├── pve-setup.yml   # Core PVE config
├── pve-install.yml # Install PVE on Debian
└── user.yml        # User management
inventory/
├── group_vars/     # Environment-specific variables
├── local.yml
├── local-dev.yml
├── remote-dev.yml
└── remote-prod.yml
```

## Collections

Roles are organized into two Ansible collections:

- **homestak.debian** - Debian-generic roles (work on any Debian host)
- **homestak.proxmox** - PVE-specific roles (depends on homestak.debian)

Playbooks reference roles using FQCN:

```yaml
roles:
  - homestak.debian.base
  - homestak.proxmox.configure
```

## Documentation

See [CLAUDE.md](CLAUDE.md) for detailed playbook information and integration testing roles.

## Third-Party Acknowledgments

| Dependency | Purpose | License |
|------------|---------|---------|
| [lae.proxmox](https://github.com/lae/ansible-role-proxmox) | Proxmox VE installation on Debian | MIT |

## Related Repos

| Repo | Purpose |
|------|---------|
| [bootstrap](https://github.com/homestak/bootstrap) | Entry point - curl\|bash setup |
| [config](https://github.com/homestak/config) | Site-specific secrets and configuration |
| [iac-driver](https://github.com/homestak-iac/iac-driver) | Orchestration engine |
| [packer](https://github.com/homestak-iac/packer) | Custom Debian cloud images |
| [tofu](https://github.com/homestak-iac/tofu) | VM provisioning with OpenTofu |

## License

Apache 2.0 - see [LICENSE](LICENSE)
