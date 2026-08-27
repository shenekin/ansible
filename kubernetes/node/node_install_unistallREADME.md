# Kubernetes Node Components

This playbook installs or uninstalls `kubelet` and `kube-proxy` on Debian/Ubuntu amd64 worker nodes. It does not install or apply a CNI plugin.

## Configuration

Edit `inventory.ini` before running the playbook. Keep versions and runtime settings there, outside the playbook.

The shared CA is read from `/root/ansible-playbook/cert/k8s/master/generated/ca.pem`. Each node uses only the matching directory `/root/ansible-playbook/cert/k8s/node/generated/<inventory-hostname>/`, containing `kubelet.pem`, `kubelet-key.pem`, `kube-proxy.pem`, and `kube-proxy-key.pem`.

The playbook copies these certificates to `/etc/kubernetes/pki/` and separately renders the kubelet kubeconfig, kube-proxy kubeconfig, and both component configuration files from templates.

## Kubernetes v1.25 runtime configuration

The kubelet is configured entirely through command-line flags in `kubelet.service.j2`; it does not load `/var/lib/kubelet/config.yaml`. In Kubernetes v1.25, a `containerRuntime` field in `KubeletConfiguration`, even an empty `containerRuntime: {}` object, overrides the command-line `--container-runtime-endpoint` value and can cause startup to fail with `missing address`.

The containerd endpoint and all other kubelet runtime settings are configured only in `kubelet.service.j2` with command-line flags.

## Prerequisites

- SSH access to every host in `inventory.ini`.
- Root SSH access or passwordless sudo.
- Debian/Ubuntu amd64 hosts.
- containerd installed and running.
- The master CA and every node's certificate files exist on the control machine.
- The Kubernetes API server is reachable from the nodes.
## Validate and install

```bash
cd /root/ansible-playbook/kubernetes/node
source ~/ansible-core/bin/activate

ansible-inventory -i inventory.ini --graph
ansible-playbook --syntax-check -i inventory.ini deploy_node_install_uninstall.yml
ansible-playbook -i inventory.ini deploy_node_install_uninstall.yml \
  -e node_install_uninstall_action=install
```

The action defaults to the value in `inventory.ini`. To preview changes:

```bash
ansible-playbook --check --diff -i inventory.ini \
  deploy_node_install_uninstall.yml \
  -e node_install_uninstall_action=install
```

## Uninstall

Uninstall services, binaries, certificates, and configuration while preserving node data:

```bash
ansible-playbook -i inventory.ini deploy_node_install_uninstall.yml \
  -e node_install_uninstall_action=uninstall
```

To also delete kubelet and CNI data, explicitly opt in:

```bash
ansible-playbook -i inventory.ini deploy_node_install_uninstall.yml \
  -e node_install_uninstall_action=uninstall \
  -e node_install_uninstall_remove_data=true
```

## Verify

```bash
ansible kubernetes_node -i inventory.ini -b -m ansible.builtin.command -a 'kubelet --version'
ansible kubernetes_node -i inventory.ini -b -m ansible.builtin.command -a 'kube-proxy --version'
ansible kubernetes_node -i inventory.ini -b -m ansible.builtin.systemd \
  -a 'name=kubelet state=started enabled=yes'
ansible kubernetes_node -i inventory.ini -b -m ansible.builtin.systemd \
  -a 'name=kube-proxy state=started enabled=yes'
```

## Troubleshooting

```bash
ansible kubernetes_node -i inventory.ini -b -m ansible.builtin.command \
  -a 'journalctl -u kubelet -n 100 --no-pager'
ansible kubernetes_node -i inventory.ini -b -m ansible.builtin.command \
  -a 'journalctl -u kube-proxy -n 100 --no-pager'
ansible kubernetes_node -i inventory.ini -b -m ansible.builtin.command \
  -a 'openssl x509 -in /etc/kubernetes/pki/kubelet.crt -noout -subject'
```
