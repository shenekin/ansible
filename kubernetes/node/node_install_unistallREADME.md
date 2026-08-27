# Kubernetes Node Installation

This playbook installs or uninstalls Kubernetes node components on Debian/Ubuntu amd64 hosts:

- kubelet `1.25.0`
- kube-proxy `1.25.0`
- Calico CNI, node, and kube-controllers `v3.26.5`

Kubernetes binaries use `https://files.m.daocloud.io`. The Calico release archive uses the DaoCloud file mirror. The supplied `calico_manifest_url` downloads the Calico Kubernetes manifest, and image references are rewritten to the China mirror in `calico_image_registry`.

## Configuration

Edit `inventory.ini` before running the playbook. Keep versions and runtime settings there, outside the playbook.

Important variables:

- `node_install_uninstall_action`: `install` or `uninstall`.
- `kubelet_version`, `kube_proxy_version`, and `calico_version`.
- `kubernetes_download_mirror`, `calico_release_mirror`, `calico_manifest_url`, `calico_manifest_node_path`, and `calico_image_registry` (default: `docker.m.daocloud.io`).
- `kube_proxy_kubeconfig_source` and `kubelet_bootstrap_kubeconfig_source`: existing kubeconfig files on the Ansible control machine.
- `kubernetes_api_server_address` and `kubernetes_api_server_port`.
- `containerd_socket`, CNI paths, and Kubernetes paths.
- `node_install_uninstall_remove_data`: defaults to `false` and protects node/container data during uninstall.

The Calico release archive is downloaded once at `calico_archive_control_path` on the Ansible control machine, then copied to `calico_archive_node_path` on every node. The Calico manifest is downloaded and finalized once at `calico_manifest_path`, then copied to `calico_manifest_node_path` on every node. Apply it once from a control-plane host after the nodes are prepared; do not apply it independently from every node. The manifest and CNI binaries both use Calico `v3.26.5`, and the manifest image references are rewritten to `docker.m.daocloud.io/calico/...`.

## Prerequisites

- SSH access to every host in `inventory.ini`.
- Root SSH access or passwordless sudo.
- Debian/Ubuntu amd64 hosts.
- containerd already installed and running.
- The kubelet bootstrap and kube-proxy kubeconfig source files exist on the control machine.
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

After node installation, apply the manifest from a control-plane host:

```bash
kubectl apply -f /root/ansible-playbook/kubernetes/node/calico.yaml
```

## Admission policy requirement

Calico uses `hostNetwork`, host-mounted CNI directories, iptables, and privileged init/main containers. A cluster policy that forbids privileged containers will reject `calico-node`; removing `privileged: true` from the manifest is not a valid fix because Calico networking will not function.

If the cluster uses Kubernetes Pod Security Admission, label the Calico namespace with the policy exception before applying the manifest:

```bash
kubectl label namespace kube-system \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/audit=privileged \
  pod-security.kubernetes.io/warn=privileged \
  --overwrite
kubectl apply -f /root/ansible-playbook/kubernetes/node/calico.yaml
```

If the same error remains, the cluster uses another admission policy such as Kyverno or Gatekeeper. An administrator must add an exception for the `calico-node` DaemonSet and its service account in `kube-system`, or allow privileged containers for that trusted system workload. Inspect the policy and webhook names with:

```bash
kubectl get validatingwebhookconfiguration,mutatingwebhookconfiguration
kubectl get clusterpolicy,clusterpolicies -A
kubectl get constrainttemplate, constraint -A
```

The action defaults to the value in `inventory.ini`. To preview changes:

```bash
ansible-playbook --check --diff -i inventory.ini \
  deploy_node_install_uninstall.yml \
  -e node_install_uninstall_action=install
```

## Uninstall

Uninstall services, binaries, CNI files, and configuration while preserving node/container data:

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
ansible kubernetes_node -i inventory.ini -b -m ansible.builtin.command -a 'calicoctl version --client'
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
  -a 'ctr -n k8s.io images list | grep calico'
```
