# Kubernetes Cluster Configuration

This directory generates Kubernetes client configuration and enables TLS bootstrap for kubelet:

- `admin.conf`: Administrator kubeconfig.
- `kube-proxy.conf`: kube-proxy kubeconfig using the `system:kube-proxy` client certificate.
- `kubelet-bootstrap.conf`: Initial kubelet kubeconfig using the bootstrap token.
- `known_tokens.csv`: Bootstrap token file for the API server.
- `bootstrap-rbac.yml`: RBAC rules that allow kubelet CSRs and automatically approve node client certificates and renewals.

## Prerequisites

1. Run `cert/k8s/master/issue_control_plane_certificates.yml` and ensure the CA and admin certificates exist in `generated/`.
2. etcd must be running with TLS enabled first. Run `cert/etcd/issue_etcd_certificates.yml`; do not run the HTTP-only `etcd/deploy_etcd.yml` when Kubernetes is configured to use HTTPS etcd endpoints.
3. Add all worker nodes to the `[kube_node]` group in `inventory.ini`.
4. Set `kubernetes_token_id` to six lowercase letters or digits and `kubernetes_token_secret` to sixteen lowercase letters or digits. Do not use the example token.
5. The API server must use `/etc/kubernetes/known_tokens.csv` installed by this playbook and load `--token-auth-file`.

## Execution Order

```bash
cd /root/ansible-playbook/cert/k8s/master
source ~/ansible-core/bin/activate
ansible-playbook -i inventory.ini issue_control_plane_certificates.yml

cd /root/ansible-playbook/cert/etcd
ansible-playbook -i inventory.ini issue_etcd_certificates.yml

cd /root/ansible-playbook/kubernetes/master/configuration
ansible-playbook --syntax-check -i inventory.ini configure_kubernetes.yml
ansible-playbook -i inventory.ini configure_kubernetes.yml
```

Generated files are stored in `generated/`. The playbook distributes `known_tokens.csv` to `/etc/kubernetes/` on every control-plane host. Redeploy or restart kube-apiserver afterward so that `--token-auth-file` takes effect.

The API server uses 10-second etcd health and readiness timeouts because the initial etcd quorum may take longer than the Kubernetes default 2 seconds to commit a request. These values can be overridden in the control-plane inventory.

## Node Usage

Copy `generated/kubelet-bootstrap.conf` to `/etc/kubernetes/bootstrap-kubelet.conf` on every node and configure kubelet to use that path at startup. Kubelet will submit a CSR; the RBAC rules allow the controller to automatically approve compliant kubelet client CSRs.

Copy `generated/kube-proxy.conf` to `/etc/kubernetes/kube-proxy.conf` on every node for kube-proxy to use.

## Security Notes

`generated/` contains private keys and the bootstrap token with `0700/0600` permissions. Do not commit it to Git or transfer it publicly. After changing the token, rerun the playbook and restart all API servers.