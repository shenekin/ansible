# etcd Mutual TLS Certificates with CFSSL

This directory contains English-only Ansible playbooks for generating and installing mutual TLS certificates for the existing three-node etcd 3.5 cluster.

## Cluster Nodes

- `etcd01`: `10.168.2.108`
- `etcd02`: `10.168.2.109`
- `etcd03`: `10.168.2.110`

The generated node certificates include each node name and IP address as SANs. They support both server authentication and client authentication for etcd peer and client connections.

## Files

- `issue_etcd_certificates.yml`: Installs the existing generated certificates and replaces the etcd systemd unit on the etcd nodes.
- `generate_etcd_certificates.yml`: Generates the CA and node certificates locally with CFSSL.
- `inventory.ini`: The scalable etcd server list and the certificate name assigned to each server.
- `ca-config.json`: CFSSL signing profile with a one-year validity period.
- `ca-csr.json`: Certificate authority request definition.
- `etcd-tls.service.j2`: TLS-enabled systemd unit for etcd.
- `generated/`: Local certificate output. This directory is ignored by Git and contains private keys.

## Prerequisites

Install CFSSL and OpenSSL on the Ansible control machine:

```bash
sudo apt-get update
sudo apt-get install -y golang-cfssl openssl
```

The generation playbook creates these files under `generated/`:

- `ca.pem`
- `etcd01.pem` and `etcd01-key.pem`
- `etcd02.pem` and `etcd02-key.pem`
- `etcd03.pem` and `etcd03-key.pem`

The installation playbook validates each certificate against the IP address in the inventory before copying it.

Verify SSH access to all etcd nodes and review the dedicated inventory:

```bash
cd /root/ansible-playbook/cert/etcd
source ~/ansible-core/bin/activate
ansible-inventory -i inventory.ini --graph
```

The deployment inventory for this directory is `inventory.ini`. Add future etcd servers there by adding one host line with its address and matching certificate name. For example:

```ini
etcd04 ansible_host=10.168.2.103 etcd_name=etcd04
```

The corresponding files must exist before deployment:

```text
generated/etcd04.pem
generated/etcd04-key.pem
```

## Generate and Install

Use the dedicated etcd inventory:

```bash
cd /root/ansible-playbook/cert/etcd
source ~/ansible-core/bin/activate
ansible-playbook --syntax-check \
  -i inventory.ini \
  generate_etcd_certificates.yml

ansible-playbook \
  -i inventory.ini \
  generate_etcd_certificates.yml

ansible-playbook --syntax-check \
  -i inventory.ini \
  issue_etcd_certificates.yml

ansible-playbook \
  -i inventory.ini \
  issue_etcd_certificates.yml
```

The playbook performs these steps:

1. Generates the CA and node certificates locally from the inventory.
2. Maps every inventory host to its configured `etcd_name` and matching certificate files.
3. Replaces the CA, node certificate, and node private key under `/etc/etcd/pki`.
4. Generates the HTTPS initial-cluster member list from the inventory.
5. Replaces `/etc/systemd/system/etcd.service` with the mutual TLS configuration.
6. Enables and restarts etcd when the certificate or configuration changes.

The CA private key remains on the control machine and is not copied to the servers.

## Verify TLS

Run the following command from an etcd node after deployment:

```bash
ETCDCTL_API=3 /usr/local/bin/etcdctl \
  --endpoints=https://10.168.2.108:2379,https://10.168.2.109:2379,https://10.168.2.110:2379 \
  --cacert=/etc/etcd/pki/ca.pem \
  --cert=/etc/etcd/pki/server.pem \
  --key=/etc/etcd/pki/server-key.pem \
  endpoint health
```

Check the service and recent logs:

```bash
systemctl status etcd
journalctl -u etcd -n 50 --no-pager
```
# Check etcd cluster status
source ~/ansible-core/bin/activate && ansible etcd01 -i inventory.ini -b -m ansible.builtin.shell -a 'ETCDCTL_API=3 /usr/local/bin/etcdctl --endpoints=https://10.168.2.108:2379,https://10.168.2.109:2379,https://10.168.2.110:2379 --cacert=/etc/etcd/pki/ca.pem --cert=/etc/etcd/pki/server.pem --key=/etc/etcd/pki/server-key.pem endpoint status -w table'
## Important Warning

Do not run this playbook against a production etcd cluster without a maintenance plan. It changes client and peer endpoints from HTTP to HTTPS and restarts etcd. Keep a backup of the existing etcd data and configuration before deployment.

The generated CA private key is required to issue future certificates. Protect the `generated/` directory and back it up securely.
