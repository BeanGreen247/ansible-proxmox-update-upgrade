> [!WARNING]
> **This repository has been consolidated.**
> Development continues at [BeanGreen247/ansible-proxmox](https://github.com/BeanGreen247/ansible-proxmox), which combines this repo with [ansible-proxmox-ve-usage-status](https://github.com/BeanGreen247/ansible-proxmox-ve-usage-status) and [proxmox-ve-vms-ansible](https://github.com/BeanGreen247/proxmox-ve-vms-ansible) into a single workspace.
> This repo is archived and will no longer receive updates.

# ansible-proxmox-update-upgrade

An Ansible playbook that mass-updates and upgrades all Proxmox LXC containers and VMs. It automatically detects the package manager on each host, runs the appropriate commands, reports a per-host upgrade summary, and reboots hosts that require it.

## Features

- **Multi-distro support** — `apt` (Debian/Ubuntu), `apk` (Alpine Linux), `dnf` (RHEL/Fedora/CentOS), `pacman` (Arch Linux); auto-detected via facts, no per-host config needed
- **Per-host upgrade summary** — prints hostname, OS version, package count, and full command output after each host
- **Dry-run before upgrade** — simulates the upgrade first to show exactly which packages will change
- **Automatic reboot handling** — reboots hosts that require it after upgrading; waits for them to come back up (timeout: 5 min)
- **Scheduled reboot support** — specific hosts (e.g. a remote-desktop VM) get a scheduled `shutdown -r 16:00` instead of an immediate reboot
- **Per-host vault secrets** — each host's root password is stored in its own encrypted `host_vars/<hostname>/vault.yml`; decrypted automatically via `~/.vault_pass.txt`
- **Bootstrap playbook** — one-time `setup-ansibleuser.yml` creates a dedicated `ansibleuser` with passwordless sudo and SSH key auth on every host, connecting as root
- **Alpine Linux hardened** — handles Alpine-specific quirks: bootstraps `python3` + `sudo` via raw task, sets correct shell (`/bin/ash`), creates user with unlocked password (`*` in shadow)
- **SSH key deployment script** — `deploy-root-key.sh` pushes your public key to root on all hosts using vault-decrypted passwords via `sshpass`
- **Performance tuned** — `forks = 20`, `pipelining = True`, SSH `ControlMaster/ControlPersist` in `ansible.cfg` for fast parallel runs
- **Idempotent** — safe to re-run; no false `changed` results when nothing actually changed
- **Unsupported host graceful skip** — unknown package managers print a warning and are skipped without failing the play

## Supported Package Managers

| Package Manager | Distros |
|---|---|
| `apt` | Debian, Ubuntu |
| `apk` | Alpine Linux |
| `dnf` | RHEL, Fedora, CentOS Stream |
| `pacman` | Arch Linux |

## Requirements

- Ansible 2.12+
- SSH access to all target hosts (key-based, as `ansibleuser`)
- `become: true` / passwordless sudo on each host (set up by `setup-ansibleuser.yml`)
- `~/.vault_pass.txt` with your vault master password
- `sshpass` on the controller for the one-time `deploy-root-key.sh` step

## Usage

After the bootstrap is done (see First-Time Setup below), run:

```bash
ansible-playbook -i inventory/hosts.ini update_upgrade.yml
```

Target a subset of hosts:

```bash
ansible-playbook -i inventory/hosts.ini update_upgrade.yml --limit lxcs
ansible-playbook -i inventory/hosts.ini update_upgrade.yml --limit lxc-prometheus
```

With an explicit vault password prompt (if not using `~/.vault_pass.txt`):

```bash
ansible-playbook -i inventory/hosts.ini update_upgrade.yml --ask-vault-pass
```

## Inventory Example

```ini
[lxcs]
lxc-prometheus  ansible_port=22 ansible_host=192.168.0.248 ansible_user=ansibleuser ansible_ssh_private_key_file=~/.ssh/id_rsa
lxc-grafana     ansible_port=22 ansible_host=192.168.0.247 ansible_user=ansibleuser ansible_ssh_private_key_file=~/.ssh/id_rsa

[vms]
vm-debian       ansible_port=22 ansible_host=192.168.0.201 ansible_user=ansibleuser ansible_ssh_private_key_file=~/.ssh/id_rsa
vm-alpine       ansible_port=22 ansible_host=192.168.0.202 ansible_user=ansibleuser ansible_ssh_private_key_file=~/.ssh/id_rsa

[all:children]
lxcs
vms
```

## What It Does Per Host

1. **Detects** the package manager automatically via gathered facts
2. **Syncs** the package index / metadata
3. **Dry-runs** the upgrade to collect the list of upgradable packages
4. **Upgrades** all packages (including `autoremove` / `autoclean` where applicable)
5. **Prints** a summary block with hostname, OS version, package count, and full command output
6. **Checks** if a reboot is required (`/var/run/reboot-required` for apt; kernel version check for apk)
7. **Reboots** if needed — immediately for most hosts; scheduled at 16:00 for designated hosts (e.g. `vm-debian-remote-desktop`)

If a host uses an unsupported package manager, it prints a warning and skips gracefully without failing the play.

## Project Structure

```
.
├── ansible.cfg
├── deploy-root-key.sh            ← run this once to push your SSH key to root on all hosts
├── group_vars
│   └── all
│       ├── example_of_main.yml   ← copy to main.yml and fill in
│       └── main.yml              ← gitignored, holds vault secrets
├── host_vars
│   └── <hostname>
│       ├── .gitkeep              ← keeps folder structure in git
│       └── vault.yml             ← gitignored, holds root password per host
├── inventory
│   └── hosts.ini
├── .gitignore
├── LICENSE
├── README.md
├── setup-ansibleuser.yml         ← run this ONCE to bootstrap all hosts
└── update_upgrade.yml            ← run this to update all hosts
```

## First-Time Setup: Bootstrap ansibleuser

Before `update_upgrade.yml` can run, `ansibleuser` must exist on every host.
The bootstrap process is a one-time operation that connects as `root` and handles everything.

### Step 1 — Generate an SSH key on your controller (skip if you already have `~/.ssh/id_rsa`)

```bash
ssh-keygen -t rsa -b 4096
# accept defaults → produces ~/.ssh/id_rsa and ~/.ssh/id_rsa.pub
```

### Step 2 — Set up the vault password file

The vault password is a single master password used to encrypt/decrypt all secrets. Choose one and store it:

```bash
echo 'your-chosen-vault-password' > ~/.vault_pass.txt
chmod 600 ~/.vault_pass.txt
```

`ansible.cfg` is already configured to use this file automatically — no `--ask-vault-pass` needed.

### Step 3 — Create a vault file for each host with its root password

```bash
mkdir -p host_vars/<hostname>
echo 'ansible_password: "the-root-password"' > host_vars/<hostname>/vault.yml
ansible-vault encrypt host_vars/<hostname>/vault.yml
```

Repeat for every host in `inventory/hosts.ini`. The `vault.yml` files are gitignored; the directory structure is kept via `.gitkeep` files.

### Step 4 — Deploy your SSH public key to root on all hosts

```bash
bash deploy-root-key.sh
```

This decrypts each host's vault file, extracts the root password, and uses `sshpass` + `ssh-copy-id` to push `~/.ssh/id_rsa.pub` to `root` on that host.

> **Note:** Requires `sshpass` — install with `sudo apt install sshpass` if missing.
> If a host fails (wrong password or `PermitRootLogin prohibit-password`), add the key manually via the Proxmox console:
> ```bash
> mkdir -p /root/.ssh && echo "your-public-key" >> /root/.ssh/authorized_keys
> chmod 700 /root/.ssh && chmod 600 /root/.ssh/authorized_keys
> ```

> **Alpine hosts** additionally need `apk add python3 sudo` in the console before bootstrapping, or the bootstrap playbook will handle it automatically if internet access is available.

### Step 5 — Run the bootstrap playbook

```bash
ansible-playbook -i inventory/hosts.ini setup-ansibleuser.yml
```

This connects as `root` (using your SSH key) and on every host:
- Creates `ansibleuser` with a home directory
- Grants passwordless sudo via `/etc/sudoers.d/ansibleuser`
- Deploys `~/.ssh/id_rsa.pub` to `ansibleuser`'s `authorized_keys`
- Verifies sudo works without a password

Test on one host first if you prefer:
```bash
ansible-playbook -i inventory/hosts.ini setup-ansibleuser.yml --limit lxc-prometheus
```

### Step 6 — Set up `group_vars/all/main.yml`

Copy the example and fill in your values:
```bash
cp group_vars/all/example_of_main.yml group_vars/all/main.yml
```

Encrypt the `ansibleuser` sudo password:
```bash
ansible-vault encrypt_string 'your_sudo_password' --name 'vault_become_password'
```

Paste the output into `main.yml` (see `example_of_main.yml` for the full format).

## Configuration

### `group_vars/all/main.yml`

```yaml
ansible_become: true
ansible_become_method: sudo
ansible_become_password: "{{ vault_become_password }}"

vault_become_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          <your encrypted string>
```

### `inventory/hosts.ini`

```ini
[lxcs]
lxc-prometheus  ansible_port=22 ansible_host=192.168.0.248 ansible_user=ansibleuser ansible_ssh_private_key_file=~/.ssh/id_rsa
lxc-grafana     ansible_port=22 ansible_host=192.168.0.247 ansible_user=ansibleuser ansible_ssh_private_key_file=~/.ssh/id_rsa

[vms]
vm-debian-hostname       ansible_port=22 ansible_host=192.168.0.201 ansible_user=ansibleuser ansible_ssh_private_key_file=~/.ssh/id_rsa
vm-alpine-hostname       ansible_port=22 ansible_host=192.168.0.202 ansible_user=ansibleuser ansible_ssh_private_key_file=~/.ssh/id_rsa

[all:children]
lxcs
vms
```

## License

MIT

---
BeanGreen247,2026