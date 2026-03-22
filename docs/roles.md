# Role Catalog

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
