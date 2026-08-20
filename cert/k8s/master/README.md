# Kubernetes 1.25 Control-Plane Certificates

This directory contains an English-only Ansible playbook for generating Kubernetes 1.25 control-plane certificates with CFSSL.

Kubelet certificates are intentionally not generated here.

## Generated Files

The playbook creates these files under `generated/`:

- `ca.pem` and `ca-key.pem`: Kubernetes cluster CA.
- `apiserver.pem` and `apiserver-key.pem`: Kubernetes API server certificate.
- `apiserver-kubelet-client.pem` and `apiserver-kubelet-client-key.pem`: API server client certificate for kubelet communication.
- `admin.pem` and `admin-key.pem`: Cluster administrator certificate.
- `controller-manager.pem` and `controller-manager-key.pem`: Controller manager client certificate.
- `scheduler.pem` and `scheduler-key.pem`: Scheduler client certificate.
- `front-proxy-ca.pem` and `front-proxy-ca-key.pem`: Aggregation layer CA.
- `front-proxy-client.pem` and `front-proxy-client-key.pem`: Aggregation layer client certificate.
- `sa.key` and `sa.pub`: Service-account signing keypair.

The API server certificate includes the configured API address, Kubernetes service names, cluster DNS name, and any extra SANs in `inventory.ini`.

## Prerequisites

Install CFSSL and OpenSSL on the Ansible control machine:

```bash
sudo apt-get update
sudo apt-get install -y golang-cfssl openssl
```

Verify the tools:

```bash
cfssl version
cfssljson --help
openssl version
```

## Configure the Inventory

Review [inventory.ini](inventory.ini) before running the playbook:

- Set `kubernetes_api_server_address` to the control-plane or load-balancer address used by clients.
- Add all required addresses to `kubernetes_api_server_extra_sans`.
- Add all required DNS names to `kubernetes_api_server_extra_dns_names`.
- Keep `kubernetes_cluster_dns` aligned with the cluster service CIDR.

The default example uses `10.168.2.100` as the API server address and `10.96.0.10` as the cluster DNS address.

## Generate the Certificates

Run from this directory:

```bash
cd /root/ansible-playbook/cert/k8s/master
source ~/ansible-core/bin/activate
ansible-playbook --syntax-check -i inventory.ini issue_control_plane_certificates.yml
ansible-playbook -i inventory.ini issue_control_plane_certificates.yml
```

The playbook generates certificates locally on the Ansible control machine. It does not copy files to servers and does not generate kubelet certificates.

## Security Notes

- Protect `generated/`, especially `ca-key.pem`, `front-proxy-ca-key.pem`, and `sa.key`.
- The generated directory is ignored by Git.
- Back up the CA private keys securely; they are required to issue replacement certificates.
- Do not reuse these CAs for unrelated environments.
