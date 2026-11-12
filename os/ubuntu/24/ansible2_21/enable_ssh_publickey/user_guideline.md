# SSH Mesh User Guide

This guide explains how to generate a separate Ed25519 key on each cluster node, distribute every node's public key to the other nodes, and verify passwordless SSH between peers.

## Files

- `ssh-mesh/inventory.ini`: cluster hosts, management addresses, and the shared remote account.
- `ssh-mesh/ssh_mesh.yml`: generates keys, collects public keys, and adds them to each account's `authorized_keys`.
- `ssh-mesh/verify_mesh.yml`: tests SSH access from every node to every other node.
- `ssh-mesh/requirements.yml`: required Ansible collections.

## 1. Prepare the Ansible environment

Run these commands from the `ansible2_21` directory with the Python environment that provides `ansible-playbook` activated:

```sh
cd /path/to/Development_Codes/ansible/os/ubuntu/24/ansible2_21
command -v ansible-playbook
command -v ansible-galaxy
ansible-playbook --version
```

Install the collections required by the playbooks:

```sh
ansible-galaxy collection install -r enable_ssh_publickey/ssh-mesh/requirements.yml
```

If `ansible-galaxy` is not found, activate the same Python environment used for Ansible or install `ansible-core` into that environment before installing the collections. The `community.crypto` collection provides `community.crypto.openssh_keypair`; `ansible.posix` provides `ansible.posix.authorized_key`.

## 2. Configure the inventory

Edit `enable_ssh_publickey/ssh-mesh/inventory.ini` and replace the example `ansible_host` addresses with the addresses that the nodes can reach over SSH. The current entries (`10.168.2.54` through `10.168.2.56`) are examples for this environment and may not match your network.

Keep `ansible_user` identical on all hosts. That account owns the generated private/public key pair and receives the cluster keys in its `authorized_keys`. The current value is `ubuntu`. Do not place plaintext passwords in the inventory; use `--ask-pass` for a shared SSH password or Ansible Vault for encrypted host-specific credentials.

The account must be able to log in to every node and write to its own home directory. TCP port 22 must be reachable from the control node to each node and between every pair of nodes. The peer addresses in `ansible_host` must therefore also be routable from the nodes themselves.

## 3. Check Ansible connectivity

Before generating keys, check that the control node can connect to every inventory host:

```sh
ansible -i enable_ssh_publickey/ssh-mesh/inventory.ini cluster -m ansible.builtin.ping
```

For password-based initial access, add `--ask-pass`:

```sh
ansible -i enable_ssh_publickey/ssh-mesh/inventory.ini cluster -m ansible.builtin.ping --ask-pass
```

Resolve any `UNREACHABLE` result before continuing. A connection timeout usually points to an incorrect address, missing route/VPN access, a firewall/security-group rule, or an SSH service that is not listening. A timeout occurs before Ansible can execute the playbook task; it is not evidence that the public key was rejected.

## 4. Build the SSH mesh

Run the playbook against the complete inventory so every host generates and registers its public key before the distribution task:

```sh
ansible-playbook -i enable_ssh_publickey/ssh-mesh/inventory.ini enable_ssh_publickey/ssh-mesh/ssh_mesh.yml
```

For password-based initial access:

```sh
ansible-playbook -i enable_ssh_publickey/ssh-mesh/inventory.ini enable_ssh_publickey/ssh-mesh/ssh_mesh.yml --ask-pass
```

Do not use `--limit` for this playbook. Its aggregation step needs the public key fact from every inventory host. The playbook is idempotent: existing key pairs are retained, and already-installed public keys are not duplicated. It does not use `exclusive: true`, so unrelated existing authorized keys are preserved.

Each node's private key stays on that node. Only the public key is read and distributed. After a successful run, each node's `ubuntu` account (or the configured `ansible_user`) can use its own private key to log in to the other cluster nodes as that same account.

## 5. Verify peer access

Run the verification playbook against the full inventory:

```sh
ansible-playbook -i enable_ssh_publickey/ssh-mesh/inventory.ini enable_ssh_publickey/ssh-mesh/verify_mesh.yml
```

The output reports each source-to-peer test as `OK` or `FAIL`. The verification uses `BatchMode=yes`, so it does not prompt for a password. It uses `StrictHostKeyChecking=accept-new`, which adds newly seen peer host keys to the remote user's `known_hosts` file.

## Troubleshooting

- **`couldn't resolve module/action 'community.crypto.openssh_keypair'`**: install the collections from `requirements.yml` in the same Ansible environment used to run `ansible-playbook`.
- **Ansible reports a host as `UNREACHABLE` with `Operation timed out`**: check control-node-to-host connectivity, the inventory address, routing/VPN, firewall rules, and whether SSH is listening on port 22. Key authentication has not yet been tested in this case.
- **The mesh task runs, but a peer test reports `FAIL`**: check node-to-node routing and firewall access to port 22, confirm the peer address is correct, and ensure the same `ansible_user` is used on every node. A `Permission denied (publickey)` response indicates an authentication/key or account issue; a timeout indicates a network/service issue.
- **SSH rejects access due to directory permissions**: ensure the remote user's home directory is not group- or world-writable and that `~/.ssh` has restrictive permissions. The playbook creates `.ssh` with mode `0700`.

## Security note

This configuration grants each cluster account SSH access to the same account on every other node. Restrict the inventory to trusted hosts and protect each node's private key. For large or dynamically changing clusters, consider SSH certificates instead of distributing every public key.

## Standalone SSH Server and Administrator Key Setup

The standalone SSH setup is separate from the mesh workflow. It configures the OpenSSH server and installs one administrator public key; it does not generate per-node keys or distribute keys between cluster peers. The existing `ssh-mesh/ssh_mesh.yml` and `ssh-mesh/verify_mesh.yml` remain dedicated to mesh key distribution and peer verification.

### Files

- `enable-ssh-access.yml`: installs and starts OpenSSH, applies the authentication settings, and installs the administrator public key.
- `vars/ssh.yml`: keeps the login account, control-node public-key path, and SSH authentication settings outside the playbook.
- `templates/sshd-auth.conf.j2`: renders the SSH server authentication settings.

### Configure

Edit `vars/ssh.yml` before running the playbook:

```yaml
ssh_login_user: ubuntu
ssh_public_key_file: "{{ lookup('env', 'HOME') }}/.ssh/id_ed25519_pve_host.pub"

ssh_password_authentication: "yes"
ssh_public_key_authentication: "yes"
ssh_keyboard_interactive_authentication: "no"
ssh_permit_root_login: "no"
```

`ssh_login_user` must already exist on the managed hosts. `ssh_public_key_file` is a path on the Ansible control node, not on the managed host; it must point to the public key matching the private key used by the administrator. Never copy or distribute the private key. Change the values in `vars/ssh.yml` to suit your policy. Enabling password authentication does not create an account password.

The playbook targets the existing `redis_nodes` group from the parent `inventory.ini`. The parent `ansible.cfg` supplies the `ubuntu` SSH connection account and its private key. Confirm the selected inventory hosts and account before applying changes.

### Install the collection

This playbook uses `ansible.posix.authorized_key`. Install the collection in the same Python environment that runs `ansible-playbook`:

```sh
ansible-galaxy collection install ansible.posix
```

### Run

Run commands from the `ansible2_21` directory. First check reachability and syntax:

```sh
ansible redis_nodes -m ansible.builtin.ping
ansible-playbook -i inventory.ini enable_ssh_publickey/enable-ssh-access.yml --syntax-check
```

Apply the playbook to one host first, replacing `hadoop01` with an active inventory alias:

```sh
ansible-playbook -i inventory.ini enable_ssh_publickey/enable-ssh-access.yml \
	--limit 10.168.2.54 \
	--ask-become-pass
```

The current parent inventory uses IP addresses as the active host names; replace `10.168.2.54` with an active inventory host or alias in your environment. If the initial SSH connection requires a password, also add `--ask-pass`. After confirming the single-host result and testing a second SSH session, apply it to the full group by omitting `--limit`:

```sh
ansible-playbook -i inventory.ini enable_ssh_publickey/enable-ssh-access.yml --ask-become-pass
```

### Verify

On a managed host, inspect the effective SSH settings:

```sh
sudo sshd -T | grep -E '^(passwordauthentication|pubkeyauthentication|kbdinteractiveauthentication|permitrootlogin) '
```

From the control node, test the matching private key in a new session:

```sh
ssh -i ~/.ssh/id_ed25519_pve_host ubuntu@10.168.2.54
```

Replace the account, address, and key path with the values for your environment. Keep the existing administrative session open until the new login succeeds. The playbook only adds the selected key and does not remove other entries from `authorized_keys`.
