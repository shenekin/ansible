# Enable SSH Password and Public-Key Authentication on Ubuntu 24.04

This guide uses an Ansible playbook to configure an Ubuntu 24.04 SSH server so that both password and public-key authentication are available. SSH server settings are deployed from a Jinja2 template, and the administrator's public key is installed on the managed account.

> Enabling password authentication does not create or set a Linux account password. The target account must already exist and have a valid password.

## 1. Prerequisites

- Ansible is installed on the control node.
- The managed host is running Ubuntu 24.04 and is reachable over the network.
- A target login account exists and can use `sudo`.
- The target account already has a password if password login is required.
- The corresponding public key is available on the control node. Do not copy the private key to the managed host.

The existing inventory and Ansible configuration are in the parent directory. Run the commands in this guide from `ansible/os/ubuntu/24/ansible2_21/` so Ansible loads `ansible.cfg` and its configured inventory, remote user, and private key.

## 2. Create the project files

The directory should contain this guide and the following Ansible files, which you will create:

```text
enable_ssh_publickey/
├── guideline.md
├── enable-ssh-access.yml
├── files/
│   └── admin_ed25519.pub
├── templates/
│   └── sshd-auth.conf.j2
└── vars/
    └── ssh.yml
```

Put the administrator's **public** key in `files/admin_ed25519.pub`. For example, a public key normally starts with `ssh-ed25519` or `ssh-rsa` and is a single line.

## 3. Define the settings

Create `vars/ssh.yml`:

```yaml
---
ssh_login_user: ubuntu
ssh_public_key_file: "{{ playbook_dir }}/files/admin_ed25519.pub"

ssh_password_authentication: "yes"
ssh_public_key_authentication: "yes"
ssh_keyboard_interactive_authentication: "no"
ssh_permit_root_login: "no"
```

Set `ssh_login_user` to the existing account that should receive the public key. Keeping root login disabled is recommended; administer the host through a regular account with `sudo`.

## 4. Add the SSH server template

Create `templates/sshd-auth.conf.j2`:

```jinja2
# Managed by Ansible. Local edits will be overwritten.
PasswordAuthentication {{ ssh_password_authentication }}
PubkeyAuthentication {{ ssh_public_key_authentication }}
KbdInteractiveAuthentication {{ ssh_keyboard_interactive_authentication }}
PermitRootLogin {{ ssh_permit_root_login }}
```

Ubuntu's OpenSSH server reads configuration snippets from `/etc/ssh/sshd_config.d/`. The playbook below installs this template as `00-ansible-auth.conf`. OpenSSH generally uses the first value it reads for an option, so a lexically early snippet can take precedence over later snippets such as cloud-init configuration. Always verify the effective settings on the target.

## 5. Add the playbook

Create `enable-ssh-access.yml`:

```yaml
---
- name: Enable SSH password and public-key authentication
  hosts: redis_nodes
  become: true
  vars_files:
    - vars/ssh.yml

  tasks:
    - name: Ensure the OpenSSH server is installed
      ansible.builtin.apt:
        name: openssh-server
        state: present
        update_cache: true

    - name: Install the SSH authentication configuration
      ansible.builtin.template:
        src: templates/sshd-auth.conf.j2
        dest: /etc/ssh/sshd_config.d/00-ansible-auth.conf
        owner: root
        group: root
        mode: "0644"
        validate: "/usr/sbin/sshd -t -f %s"
      notify:
        - Validate SSH server configuration
        - Reload SSH server

    - name: Install the administrator public key
      ansible.posix.authorized_key:
        user: "{{ ssh_login_user }}"
        key: "{{ lookup('ansible.builtin.file', ssh_public_key_file) }}"
        state: present

    - name: Ensure SSH server is enabled and running
      ansible.builtin.service:
        name: ssh
        enabled: true
        state: started

  handlers:
    - name: Validate SSH server configuration
      ansible.builtin.command: /usr/sbin/sshd -t
      changed_when: false

    - name: Reload SSH server
      ansible.builtin.service:
        name: ssh
        state: reloaded
```

The playbook uses `ansible.posix.authorized_key`. Install the collection on the control node if it is not already available:

```sh
ansible-galaxy collection install ansible.posix
```

## 6. Check the inventory

The existing `inventory.ini` defines the `redis_nodes` group, and `ansible.cfg` sets `ubuntu` as the remote user. Confirm that the selected host is reachable and that `ssh_login_user` is the account that should receive the public key. Do not add a duplicate group if the existing inventory is suitable.

The playbook targets the whole `redis_nodes` group by default. Use `--limit hadoop01` to test one host first, replacing `hadoop01` with the inventory alias you intend to manage.

## 7. Run the playbook

From the `ansible2_21/` directory, test on one host first. The existing `ansible.cfg` supplies the inventory, remote user, and configured private key; prompt for the `sudo` password:

```sh
ansible-playbook -i inventory.ini enable_ssh_publickey/enable-ssh-access.yml \
  --limit hadoop01 \
  --ask-become-pass
```

For an initial connection using an SSH password instead of the configured key, prompt for the SSH and `sudo` passwords:

```sh
ansible-playbook -i inventory.ini enable_ssh_publickey/enable-ssh-access.yml \
  --limit hadoop01 \
  --ask-pass \
  --ask-become-pass
```

After the single-host test succeeds, omit `--limit hadoop01` to apply the playbook to the full `redis_nodes` group. Do not put plaintext passwords in the playbook or commit them to the inventory. If inventory variables such as `ansible_password` or `ansible_become_password` are required, protect them with Ansible Vault.

## 8. Verify the result

On the managed host, inspect the effective SSH server settings:

```sh
sudo sshd -T | grep -E '^(passwordauthentication|pubkeyauthentication|kbdinteractiveauthentication|permitrootlogin) '
```

Expected values include `passwordauthentication yes` and `pubkeyauthentication yes`. `permitrootlogin no` is expected with the recommended setting above.

From the control node, test public-key access in a new terminal session:

```sh
ssh -i ~/.ssh/id_ed25519_pve_host ubuntu@10.168.2.54
```

To test password authentication specifically, use a temporary client invocation that disables public-key authentication:

```sh
ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password ubuntu@10.168.2.54
```

Replace the account, address, and private-key path with your actual values. The configured private-key path is shown as an example based on the current `ansible.cfg`; use the matching key for the public key installed by the playbook.

## 9. Operational cautions

- Keep the current administrative SSH session open until a second session has successfully connected with the intended method.
- Confirm that firewall rules and any cloud security groups allow inbound TCP port 22 before testing.
- Password authentication increases exposure to password guessing. Use strong, unique passwords and restrict network access where possible. If password login is not required, set `ssh_password_authentication: "no"` after confirming key-based access works.
- If a setting does not match the expected value, inspect `/etc/ssh/sshd_config` and every file in `/etc/ssh/sshd_config.d/`. Also check for `Match` blocks, which can change effective settings for particular users or connections.
- To remove a key managed by this playbook, change the `authorized_key` task to `state: absent` for that key. Do not remove an administrator's only working access path before testing an alternative.