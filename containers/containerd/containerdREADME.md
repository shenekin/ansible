# Install containerd

This playbook installs or uninstalls the following components on Debian or Ubuntu amd64 hosts:

- containerd `1.6.24`
- runc `1.1.10`
- crictl `1.25.0`

All downloads use `https://files.m.daocloud.io` as a GitHub mirror.

## Prerequisites

- Ansible on the control machine in `~/ansible-core`.
- SSH access to the target host.
- Root SSH access, or passwordless sudo.
- A Debian-family amd64 target.

Edit `inventory.ini` before deployment. It contains the target hosts and all operator configuration, including:

- `containerd_action`: `install` or `uninstall`.
- `containerd_version`, `runc_version`, and `crictl_version`.
- `download_mirror` and installation paths.
- `containerd_configure_systemd_cgroup`.
- `containerd_remove_data`, which defaults to `false` during uninstall.

## Validate and deploy

Run these commands from this directory:

```bash
cd /root/ansible-playbook/containers/containerd
source ~/ansible-core/bin/activate

ansible-inventory -i inventory.ini --graph
ansible-playbook --syntax-check -i inventory.ini deploy_containerd.yml
ansible-playbook -i inventory.ini deploy_containerd.yml
```

The action is read from `inventory.ini`. To override it for one run, install with:

```bash
ansible-playbook -i inventory.ini deploy_containerd.yml \
  -e containerd_action=install
```

Uninstall containerd, runc, and crictl with:

```bash
ansible-playbook -i inventory.ini deploy_containerd.yml \
  -e containerd_action=uninstall
```

Uninstall and also delete `/var/lib/containerd` data:

```bash
ansible-playbook -i inventory.ini deploy_containerd.yml \
  -e containerd_action=uninstall \
  -e containerd_remove_data=true
```

For a preview without changing the hosts:

```bash
ansible-playbook --check --diff -i inventory.ini deploy_containerd.yml
```

## Verify after deployment

```bash
ansible containerd -i inventory.ini -b -m ansible.builtin.command \
  -a 'containerd --version'
ansible containerd -i inventory.ini -b -m ansible.builtin.command \
  -a 'runc --version'
ansible containerd -i inventory.ini -b -m ansible.builtin.command \
  -a 'crictl --version'
ansible containerd -i inventory.ini -b -m ansible.builtin.systemd \
  -a 'name=containerd state=started enabled=yes'
```

The playbook writes `/etc/containerd/config.toml` with `SystemdCgroup = true`, installs the service at `/etc/systemd/system/containerd.service`, and enables containerd at boot.

## Troubleshooting

View the containerd service log:

```bash
ansible containerd -i inventory.ini -b -m ansible.builtin.command \
  -a 'journalctl -u containerd -n 100 --no-pager'
```

Check the mirror URLs if a download fails. The playbook intentionally stops on unsupported operating systems or non-amd64 hosts.
