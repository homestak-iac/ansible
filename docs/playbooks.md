# Playbooks

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
