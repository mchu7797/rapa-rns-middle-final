# RNS Final — Cisco IOS Network Automation

Ansible project that configures Cisco IOS routers (hostname, interfaces, L3,
OSPF, BGP) using the `cisco.ios` resource modules.

## Layout

```
ansible.cfg                     # inventory path, roles_path, connection tuning
inventory.yaml                  # device groups (ATNT, Transit, Google, KT, LG)
collections/requirements.yml    # required Ansible collections
group_vars/all/
  ├── vars.yaml                 # connection vars + references to vault_*
  └── vault.yaml                # secrets (ENCRYPT with ansible-vault)
host_vars/R*.yaml               # per-router interface/OSPF/BGP config
roles/ios_config/               # reusable role that applies the resources
playbooks/
  ├── apply_all.yaml            # push config  (state: merged)
  └── render_all.yaml           # dry-run, print rendered CLI (state: rendered)
```

## Setup

```bash
ansible-galaxy collection install -r collections/requirements.yml
ansible-vault encrypt group_vars/all/vault.yaml   # first time only
```

## Usage

```bash
# Dry-run: render the CLI commands without touching devices
ansible-playbook playbooks/render_all.yaml --ask-vault-pass

# Apply configuration to all devices
ansible-playbook playbooks/apply_all.yaml --ask-vault-pass

# Target a single group / host
ansible-playbook playbooks/apply_all.yaml --limit ATNT --ask-vault-pass
```
