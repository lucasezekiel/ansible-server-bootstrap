# Ansible Server Bootstrap

An Ansible playbook that automates the bootstrap of a freshly installed Ubuntu server: baseline configuration, security hardening, a Docker installation and an optional monitoring lab.

> **Lab only — review before running.** This playbook changes users, SSH, firewall and Docker settings on the target. It is not a production-ready security baseline. Use a disposable test server with console access; running it against the wrong host can lock you out or expose services.

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
- Configures an early SSH drop-in to disable password and keyboard-interactive authentication, and validates the global effective settings before the SSH restart
- Disables remote root login
- Configures UFW with default deny/allow policies (incoming/outgoing), permitting SSH

**`docker` role**

- Adds Docker's official GPG key and apt repository
- Installs Docker Engine, CLI, containerd, Buildx and Compose plugins
- Enables and starts the Docker service
- Adds the administrative user to the `docker` group only if `docker_grant_admin_group` is enabled

**`monitoring` role** (disabled by default)

- Installs Node Exporter from an upstream archive and runs it as a systemd service
- Deploys Prometheus and Grafana with Docker Compose, using host networking
- Binds Grafana (`3000/tcp`), Prometheus (`9090/tcp`) and Node Exporter (`9100/tcp`) to loopback; does not create firewall rules for them

### Security limitations

- Docker group membership provides root-equivalent access and is opt-in. Administrative commands without it normally require `sudo` on the target.
- The playbook does not set an administrator password. On systems where sudo requires the user's password, provision it separately through a secure channel before relying on that account for sudo. Keep the original administrative access until the new account is verified.
- Monitoring uses host networking and `latest` image tags. Loopback binding is intended to limit exposure, but is not yet verified by an integration test. Review Grafana's initial credentials and effective network exposure before use; do not expose this lab to untrusted networks.
- Docker networking can bypass expected UFW filtering for published ports; verify effective network exposure rather than relying on UFW rules alone.
- SSH hardening can prevent remote access if the public key or target configuration is incorrect. Keep a separate administrative session and console access available while testing.
- The SSH validation checks global effective settings. A fresh SSH connection was tested for the lab administrator, but other host- or user-specific `Match` rules must also be reviewed and tested.
- The SSH configuration depends on the target's `/etc/ssh/sshd_config` including `/etc/ssh/sshd_config.d/*.conf`. If an earlier setting takes precedence, validation fails before the restart; review and correct the file before a later restart.
- The Node Exporter archive is downloaded without a checksum verification step. Verify artifacts before using them in a real environment.
- Disabling an option or removing a task does not undo changes applied by an earlier version. In particular, review any existing `NOPASSWD` sudoers rules, Docker group membership, monitoring services and UFW rules separately before changing a previously configured target.
- Docker APT sources now use the deb822 format. Before running any roles, the playbook stops if `/etc/apt/sources.list.d/docker.list` or `docker.sources` exists; review and migrate older or independently managed sources manually. Other duplicate sources may also exist under different names. No existing source is removed automatically.

## Project structure

```
├── inventory/hosts.yml       # target server(s) definition
├── group_vars/all.yml        # variables (admin user, packages, timezone, SSH key path)
├── roles/
│   ├── common/tasks/main.yml
│   ├── security/
│   │   ├── tasks/main.yml
│   │   └── handlers/main.yml # SSH restart, only triggered on real changes
│   ├── docker/tasks/main.yml
│   └── monitoring/           # Node Exporter, Prometheus and Grafana
├── site.yml                  # entry playbook
├── requirements.yml          # pinned Ansible collection versions
├── .github/workflows/ci.yml  # static checks
├── .gitignore
└── LICENSE
```

## Requirements

- Ansible collections `ansible.posix`, `community.general` and `community.docker` on the control node (versions in `requirements.yml`)
- The static checks are configured for Ansible Core 2.22.0 and ansible-lint 26.11.0; see the scope of the VM test below
- A disposable target server reachable via SSH (tested against Ubuntu 22.04.5 LTS on QEMU/KVM; an earlier version was tested with Multipass)
- Your own SSH key pair (defaults to `~/.ssh/id_ed25519`)

## Usage

**Before running:** `inventory/hosts.yml` uses `192.0.2.10`, an address reserved for documentation, not a configured target. Prepare your own inventory and check that it points only to the intended lab host. Set `ansible_user` to the account used for the initial SSH connection, `admin_user` to the administrator the playbook should manage, and `ssh_pubkey_path` to the public key you can use to reconnect. The defaults are examples, not universal values. Keep console access and a recovery point available.

Install the declared collections on the control node:

```bash
ansible-galaxy collection install -r requirements.yml
```

```bash
# Verify connectivity
ansible -i inventory/hosts.yml all -m ping

# Verify which hosts the playbook would select
ansible-playbook -i inventory/hosts.yml site.yml --list-hosts

# Simulate supported tasks; check mode is not a guarantee of zero side effects
ansible-playbook -i inventory/hosts.yml site.yml --check --ask-become-pass

# Apply for real
ansible-playbook -i inventory/hosts.yml site.yml --ask-become-pass
```

Docker group membership and monitoring are disabled by default in `group_vars/all.yml`. To enable either for a disposable lab, set `docker_grant_admin_group` or `enable_monitoring` to `true` after reviewing the security limitations above. If group membership is enabled, a new SSH session is needed for it to take effect.

## Static validation

The GitHub Actions workflow is configured to check playbook syntax and run ansible-lint on pushes and pull requests; its first remote run is still pending. These checks do not connect to a target or verify deployment behavior. Run the same checks locally after installing the required tools and collections:

```bash
ansible-playbook -i inventory/hosts.yml site.yml --syntax-check
ansible-lint --offline --nocolor site.yml
```

## Lab test and idempotency

The current deb822 revision was applied on an Ubuntu Server 22.04.5 QEMU/KVM VM restored to a pre-playbook baseline, using Ansible Core 2.22.0. The VM already had its administrator account and SSH key configured. The first run completed with:

```
ubuntu_ansible_lab : ok=26  changed=10  unreachable=0  failed=0  skipped=10
```

Afterward, the deb822 Docker source was present, Docker was active, and a fresh SSH connection with the lab key succeeded. For the lab user, the server offered only public-key authentication. Monitoring and Docker group membership were disabled and skipped. A repeat run within the APT cache validity window completed with:

```
ubuntu_ansible_lab : ok=25  changed=0  unreachable=0  failed=0  skipped=10
```

For comparison, the preceding revision was repeated on an already provisioned VM within the APT cache validity window:

```
PLAY RECAP *********************************************************
ubuntu_ansible_lab : ok=22  changed=0  unreachable=0  failed=0  skipped=10
```

These results cover the particular lab state and configuration; they do not prove behavior on every host, migration of an existing Docker repository, or the optional monitoring role.

## Design decisions

- **UFW instead of raw iptables**: prioritizes readability and maintainability over granular control, which fits the scope of this project.
- **Handler for the SSH restart**: the service is restarted only when the managed SSH drop-in changes, after checking the effective global configuration.
- **Bounded apt cache refresh (`cache_valid_time: 3600`)**: the apt index is only refreshed if the last update is more than an hour old. This is a deliberate, time-windowed form of idempotency rather than a strict one — refreshing on every run would report `changed` even when no new packages exist, adding noise without value.
- **Docker installed from the official upstream repository** rather than the distro's default packages, to track upstream releases directly.

## Known limitations

- **SSH key task under `--check`**: the task that copies the SSH public key depends on the admin user already existing. In simulation mode, Ansible never creates the user for real, so this specific task fails under `--check` even though the real run completes without issues.
- **Docker role under `--check`**: repository configuration depends on `python3-debian` and the key file being present, and package installation depends on the repository being configured. On a fresh target, simulation may fail because earlier tasks do not create these prerequisites in check mode; the current deb822 version has not been tested in check mode on a clean VM.

Both are instances of the same underlying pattern: **chained tasks that depend on a previous task's real-world side effect will report false failures under `--check`.** This is a documented Ansible behavior, not a bug in this playbook.

## Next improvements

- Ansible Vault for sensitive variables
- Verify loopback-only monitoring in an integration test, pin images and verify downloaded artifacts
- Design and test an explicit migration procedure for previously configured hosts
- Automated testing with Molecule
- Verify the GitHub Actions workflow after publishing; add integration tests for real deployments
- Multi-host inventory example (staging/production groups)

## License

MIT — see [LICENSE](LICENSE)
