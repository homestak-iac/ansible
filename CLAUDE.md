# Ansible - Ansible Configuration for Proxmox

Ansible playbooks for configuring fresh Proxmox VE installations and installing PVE on Debian.

## Ecosystem Context

This repo is part of the homestak polyrepo workspace. For project architecture,
development lifecycle, sprint/release process, and cross-repo conventions, see:

- `~/homestak/dev/meta/CLAUDE.md` — primary reference
- `docs/lifecycle/` in meta — 7-phase development process
- `docs/CLAUDE-GUIDELINES.md` in meta — documentation standards

When working in a scoped session (this repo only), follow the same sprint/release
process defined in meta. Use `/session save` before context compaction and
`/session resume` to restore state in new sessions.

### Agent Boundaries

This agent operates within the following constraints:

- Opens PRs via `homestak-bot`; never merges without human approval
- Runs lint and validation tools only; never executes infrastructure operations
- Never runs `ansible-playbook` or modifies live host configuration

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

# Lint
ansible-lint

# Install dependencies
make install-deps
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
├── docs/                 # Detailed documentation
│   ├── roles.md          # Role catalog, variables, networking tasks
│   └── playbooks.md      # Execution models, playbook details, installation
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

Two collections organized by scope:

| Collection | Roles | Purpose |
|------------|-------|---------|
| `homestak.debian` | base, users, security, iac_tools | Debian-generic (any host) |
| `homestak.proxmox` | install, configure, networking, api_token | PVE-specific (depends on debian) |

Playbooks reference roles by FQCN (e.g., `homestak.debian.base`, `homestak.proxmox.configure`).

## Documentation

@docs/roles.md
@docs/playbooks.md

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

## GitHub Repository

- https://github.com/homestak-iac/ansible

## License

Apache 2.0 - see [LICENSE](LICENSE)
