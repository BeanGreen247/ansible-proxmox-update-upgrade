# ansible-proxmox-update-upgrade

An Ansible playbook that automatically detects the package manager on each target host (LXC container or VM) and runs the appropriate update and upgrade commands. Prints the number of upgraded packages and full command output per host.

## Supported Package Managers

| Package Manager | Distros |
|---|---|
| `apt` | Debian, Ubuntu |
| `apk` | Alpine Linux |
| `dnf` | RHEL, Fedora, CentOS Stream |
| `pacman` | Arch Linux |

Detection is handled automatically via Ansible's `ansible_pkg_mgr` fact — no manual configuration needed per host.

## Requirements

- Ansible 2.12+
- SSH access to all target hosts
- `become: true` privileges (sudo) on each host

## Usage

After the bootstrap is done (see First-Time Setup below), run:

```bash
ansible-playbook -i inventory/hosts.ini update_upgrade.yml
```

With explicit vault password prompt:

```bash
ansible-playbook -i inventory/hosts.ini update_upgrade.yml --ask-vault-pass
```

Target a subset of hosts:

```bash
ansible-playbook -i inventory/hosts.ini update_upgrade.yml --limit lxcs
ansible-playbook -i inventory/hosts.ini update_upgrade.yml --limit lxc-prometheus
```

## Inventory Example

```ini
[lxc_containers]
192.168.1.10
192.168.1.11

[vms]
192.168.1.20
192.168.1.21

[all:vars]
ansible_user=root
```

## What It Does Per Host

1. **Detects** the package manager automatically via gathered facts
2. **Syncs** the package index / metadata
3. **Dry-runs** the upgrade to collect the list of upgradable packages
4. **Upgrades** all packages
5. **Prints** a summary block with:
   - Hostname and OS version
   - Number of packages upgraded
   - Full output of each command

If a host uses an unsupported package manager, it prints a warning and skips gracefully without failing the play.

## Output Example

```
TASK [[apt] Show upgrade summary] *******************************************
ok: [192.168.1.10] => {
    "msg": "Host              : 192.168.1.10\nOS                : Debian 12\nPackages upgraded : 14\n\n--- apt-get dist-upgrade output (simulated) ---\n..."
}
```

## Project Structure

```
.
├── ansible.cfg
├── group_vars
│   └── all
│       ├── example_of_main.yml   ← copy to main.yml and fill in
│       └── main.yml              ← gitignored, holds vault secrets
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
Use the included bootstrap playbook — it connects as `root` and handles everything automatically.

### Step 1 — Generate an SSH key on your controller (if you don't have one)

```bash
ssh-keygen -t rsa -b 4096 -C "ansibleuser@controller"
# accept defaults → produces ~/.ssh/id_rsa and ~/.ssh/id_rsa.pub
```

### Step 2 — Run the bootstrap playbook as root

```bash
ansible-playbook -i inventory/hosts.ini setup-ansibleuser.yml -u root --ask-pass
```

This will, on every host:
- Create `ansibleuser` with a home directory
- Grant passwordless sudo via `/etc/sudoers.d/ansibleuser`
- Deploy your `~/.ssh/id_rsa.pub` to `ansibleuser`'s `authorized_keys`

To target a single host first (recommended for testing):

```bash
ansible-playbook -i inventory/hosts.ini setup-ansibleuser.yml \
  -u root --ask-pass --limit lxc-prometheus
```

### Step 3 — Encrypt the sudo password with Ansible Vault

```bash
ansible-vault encrypt_string --name 'vault_become_password' --vault-id default@prompt 'your_sudo_password'
```

Paste the output into `group_vars/all/main.yml` (see `example_of_main.yml` for the format).

### Step 4 — Optionally enable passwordless vault execution

```bash
echo 'yourVaultPassword' > ~/.vault_pass.txt
chmod 600 ~/.vault_pass.txt
echo 'export ANSIBLE_VAULT_PASSWORD_FILE=~/.vault_pass.txt' >> ~/.bashrc
source ~/.bashrc
```

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
lxc-debian  ansible_port=22 ansible_host=192.168.0.101 ansible_user=ansibleuser ansible_ssh_private_key_file=~/.ssh/id_rsa
lxc-ubuntu  ansible_port=22 ansible_host=192.168.0.102 ansible_user=ansibleuser ansible_ssh_private_key_file=~/.ssh/id_rsa
lxc-alpine  ansible_port=22 ansible_host=192.168.0.103 ansible_user=ansibleuser ansible_ssh_private_key_file=~/.ssh/id_rsa

[vms]
vm-debian   ansible_port=22 ansible_host=192.168.0.201 ansible_user=ansibleuser ansible_ssh_private_key_file=~/.ssh/id_rsa
vm-alpine   ansible_port=22 ansible_host=192.168.0.202 ansible_user=ansibleuser ansible_ssh_private_key_file=~/.ssh/id_rsa

[all:children]
lxcs
vms
```

## License

MIT
