# Ansible Server Bootstrap

An Ansible playbook that automates the security bootstrap of a freshly installed Ubuntu server: creating an administrative user with SSH key-based access, disabling password authentication and root login, and configuring a firewall with UFW.

## What it does

The `security` role idempotently applies:

- Creation of an administrative user with sudo privileges
- Copying the operator's SSH public key to the new user
- Disabling SSH password authentication
- Disabling remote root login
- UFW firewall with default deny/allow policies (incoming/outgoing), permitting SSH only

## Project structure

```
├── inventory/hosts.yml     # target server(s) definition
├── group_vars/all.yml      # variables (admin user, SSH key path)
├── roles/security/
│   ├── tasks/main.yml      # hardening tasks
│   └── handlers/main.yml   # SSH restart, only triggered on real changes
└── site.yml                # entry playbook
```

## Requirements

- Ansible ≥ 2.16 on the control node
- A target server reachable via SSH (tested against Ubuntu 22.04 LTS via Multipass)
- Your own SSH key pair (defaults to `~/.ssh/id_ed25519`)

## Usage

```bash
# Verify connectivity
ansible -i inventory/hosts.yml all -m ping

# Simulate changes without applying them
ansible-playbook -i inventory/hosts.yml site.yml --check --diff

# Apply for real
ansible-playbook -i inventory/hosts.yml site.yml
```

## Idempotency proof

Running the playbook a second time, with no changes on the target system, produces:

```
PLAY RECAP *********************************************************
ansible-target : ok=10  changed=0  unreachable=0  failed=0  skipped=0
```

`changed=0` confirms the playbook is idempotent: re-applying it has no additional effect once the system has reached the desired state.

## Design decisions

- **UFW instead of raw iptables**: prioritizes readability and maintainability over granular control, which fits the scope of this project.
- **Handlers for the SSH restart**: the service is only restarted if `sshd_config` actually changed, avoiding unnecessary restarts.
- **Known `--check` limitation**: the task that copies the SSH key depends on the admin user already existing. In simulation mode, Ansible never creates the user for real, so that specific task fails under `--check` even though the real run completes without issues. This is a documented Ansible limitation with chained tasks, not a bug in the playbook.

## Next improvements

- `common` role (base packages, timezone, automatic updates)
- `docker` role
- Ansible Vault for sensitive variables
- Automated testing with Molecule

## License

MIT
