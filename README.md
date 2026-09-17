# Ansible Server Bootstrap

An Ansible playbook that automates the bootstrap of a freshly installed Ubuntu server: baseline configuration, security hardening, and a ready-to-use Docker installation.

## What it does

**`common` role**

- Updates the apt package index (bounded by a 1-hour cache window)
- Installs base packages (`git`, `curl`, `wget`, `htop`, `unzip`, `ca-certificates`)
- Sets the system timezone
- Ensures time synchronization is active
- Enables automatic security updates via `unattended-upgrades`

**`security` role**

- Creates an administrative user with sudo privileges
- Copies the operator's SSH public key to the new user
- Disables SSH password authentication
- Disables remote root login
- Configures UFW with default deny/allow policies (incoming/outgoing), permitting SSH only

**`docker` role**

- Adds Docker's official GPG key and apt repository
- Installs Docker Engine, CLI, containerd, Buildx and Compose plugins
- Enables and starts the Docker service
- Adds the administrative user to the `docker` group (passwordless Docker usage)

## Project structure

```
├── inventory/hosts.yml       # target server(s) definition
├── group_vars/all.yml        # variables (admin user, packages, timezone, SSH key path)
├── roles/
│   ├── common/tasks/main.yml
│   ├── security/
│   │   ├── tasks/main.yml
│   │   └── handlers/main.yml # SSH restart, only triggered on real changes
│   └── docker/tasks/main.yml
├── site.yml                  # entry playbook
├── .gitignore
└── LICENSE
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

After the `docker` role runs for the first time, log out and start a new SSH session before using Docker without `sudo` — group membership changes don't apply to already-open sessions.

## Idempotency proof

Running the full playbook (all three roles) a second time, with no changes on the target system, produces:

```
PLAY RECAP *********************************************************
ansible-target : ok=22  changed=0  unreachable=0  failed=0  skipped=0
```

`changed=0` confirms the playbook is idempotent: re-applying it has no additional effect once the system has reached the desired state.

## Design decisions

- **UFW instead of raw iptables**: prioritizes readability and maintainability over granular control, which fits the scope of this project.
- **Handlers for the SSH restart**: the service is only restarted if `sshd_config` actually changed, avoiding unnecessary restarts.
- **Bounded apt cache refresh (`cache_valid_time: 3600`)**: the apt index is only refreshed if the last update is more than an hour old. This is a deliberate, time-windowed form of idempotency rather than a strict one — refreshing on every run would report `changed` even when no new packages exist, adding noise without value.
- **Docker installed from the official upstream repository** rather than the distro's default packages, to track upstream releases directly.

## Known limitations

- **SSH key task under `--check`**: the task that copies the SSH public key depends on the admin user already existing. In simulation mode, Ansible never creates the user for real, so this specific task fails under `--check` even though the real run completes without issues.
- **Docker install task under `--check`**: the task that installs Docker packages depends on the apt repository file written by a previous task. Since `--check` mode never writes that file for real, `apt` can't resolve `docker-ce` in simulation mode, even though the real run installs it correctly.

Both are instances of the same underlying pattern: **chained tasks that depend on a previous task's real-world side effect will report false failures under `--check`.** This is a documented Ansible behavior, not a bug in this playbook.

## Next improvements

- Ansible Vault for sensitive variables
- Automated testing with Molecule
- GitHub Actions workflow to lint and syntax-check the playbook on every push
- Multi-host inventory example (staging/production groups)

## License

MIT — see [LICENSE](LICENSE)
