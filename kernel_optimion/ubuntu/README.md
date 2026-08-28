# Ubuntu 24.04 host preparation for Kubernetes 1.25

This directory contains an Ansible playbook for preparing Ubuntu 24.04 hosts before installing Kubernetes 1.25. It configures the kernel prerequisites documented by Kubernetes:

- Loads and persists the `overlay` and `br_netfilter` kernel modules.
- Enables bridge traffic visibility for iptables and ip6tables.
- Enables IPv4 forwarding.
- Disables active swap and comments swap entries in `/etc/fstab`.

This is host preparation only. It does not install Kubernetes, a container runtime, CNI plugins, or firewall rules.

## Compatibility note

Kubernetes 1.25 is end-of-life. Ubuntu 24.04 is newer than the operating systems commonly used and tested with Kubernetes 1.25. Confirm the exact Kubernetes, container runtime, and kernel combination in your environment before production use. This playbook validates that targets are Ubuntu 24.x, but it cannot guarantee compatibility between an end-of-life Kubernetes release and Ubuntu 24.04.

## Files

- `inventory.ini`: target hosts and SSH settings.
- `group_vars/all.yml`: configurable host packages, kernel modules, swap behavior, and reboot behavior.
- `prepare_kubernetes_ubuntu.yml`: execution logic.
- `templates/modules-load.conf.j2`: persistent module configuration.
- `templates/99-kubernetes.conf.j2`: persistent sysctl configuration.

Configuration and execution are intentionally separate. Routine changes should be made in `group_vars/all.yml`; task behavior belongs in the playbook, and generated host configuration belongs in `templates/`.

## Requirements

- Ansible installed on the control machine.
- SSH access to every target host.
- The SSH user can use passwordless `sudo`, or connect directly as `root`.
- Python 3 available on each target host.
- A maintenance window if a reboot is later enabled.

## Configure the inventory

Edit `inventory.ini` and uncomment or add the actual Kubernetes nodes:

```ini
[kubernetes_nodes]
node01 ansible_host=10.168.2.110 ansible_user=ubuntu
node02 ansible_host=10.168.2.111 ansible_user=ubuntu
```

Do not place passwords in this file. Use Ansible Vault or SSH keys for credentials.

## Review configuration

Edit `group_vars/all.yml` before execution. The default values are the Kubernetes networking prerequisites. The playbook does not reboot hosts unless this is explicitly changed:

```yaml
kubernetes_reboot_after_configuration: false
```

Set it to `true` only when the resulting reboot is acceptable. A reboot is normally unnecessary for the changes in this playbook because modules and sysctls are applied immediately.

## Validate and run

The following commands are provided for the operator to run. They are not executed by this project setup:

```bash
cd /root/ansible-playbook/kernel_optimion/ubuntu

# Optional: activate the repository's Ansible environment.
source ~/ansible-core/bin/activate

# Confirm that the intended hosts are in the inventory.
ansible-inventory -i inventory.ini --graph

# Validate playbook syntax without connecting to hosts.
ansible-playbook --syntax-check -i inventory.ini prepare_kubernetes_ubuntu.yml

# Preview changes without modifying hosts.
ansible-playbook --check --diff -i inventory.ini prepare_kubernetes_ubuntu.yml

# Apply the host preparation.
ansible-playbook -i inventory.ini prepare_kubernetes_ubuntu.yml
```

To supply a sudo password interactively, add `-K` to the relevant `ansible-playbook` command. Do not put the password in the playbook or inventory.

## Verify the result

Run these commands on a target host after the playbook completes:

```bash
lsmod | grep -E '^(overlay|br_netfilter)'

sysctl net.bridge.bridge-nf-call-iptables \
       net.bridge.bridge-nf-call-ip6tables \
       net.ipv4.ip_forward

swapon --show
```
modprobe br_netfilter
cat > /etc/sysctl.d/k8s.conf <<EOF
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sysctl -p /etc/sysctl.d/k8s.conf


`swapon --show` should return no active swap devices. The sysctl values should each be `1`.

## Extend or modify it

- Add or remove host packages in `group_vars/all.yml` under `kubernetes_host_packages`.
- Add or remove modules in `group_vars/all.yml` under `kubernetes_kernel_modules`, then keep `templates/modules-load.conf.j2` aligned with that list.
- Add or change sysctl keys in `templates/99-kubernetes.conf.j2`; run `sysctl --system` after applying the playbook.
- Keep persistent module-file formatting in `templates/modules-load.conf.j2`.
- Keep persistent sysctl-file formatting in `templates/99-kubernetes.conf.j2`.
- Add operational steps or validation to `prepare_kubernetes_ubuntu.yml`.
- Run `--syntax-check` and `--check --diff` after every change before applying it.

When adding a setting, verify that it is required by the Kubernetes version and container runtime you actually use. Avoid copying tuning values from unrelated distributions or newer Kubernetes releases without testing them.

## Official references

- Kubernetes container runtimes: https://kubernetes.io/docs/setup/production-environment/container-runtimes/
- Kubernetes installation tools: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
- Kubernetes network plugin requirements: https://kubernetes.io/docs/setup/production-environment/container-runtimes/#:~:text=Ensure%20net%2Ebridge%2Ebridge%2Dnf%2Dcall%2Diptables%20is%20set%20to%201
