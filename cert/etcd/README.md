# etcd Mutual TLS Certificates with CFSSL

This directory contains an English-only Ansible playbook that creates and installs mutual TLS certificates for the existing three-node etcd 3.5 cluster.

## Cluster Nodes

- `etcd01`: `10.168.2.100`
- `etcd02`: `10.168.2.101`
- `etcd03`: `10.168.2.102`

The generated node certificates include each node name and IP address as SANs. They support both server authentication and client authentication for etcd peer and client connections.

## Files

- `issue_etcd_certificates.yml`: Generates certificates locally and installs them on the etcd nodes.
- `ca-config.json`: CFSSL signing profile with a one-year validity period.
- `ca-csr.json`: Certificate authority request definition.
- `etcd-tls.service.j2`: TLS-enabled systemd unit for etcd.
- `generated/`: Local certificate output. This directory is ignored by Git and contains private keys.

## Prerequisites

Install `cfssl` and `cfssljson` on the Ansible control machine and make sure both commands are in `PATH`:

On Ubuntu 24.04:

```bash
sudo apt-get update
sudo apt-get install -y golang-cfssl
```

```bash
cfssl version
cfssljson --help
```

Also verify SSH access to all etcd nodes and review the existing inventory:

```bash
cd /root/ansible-playbook/cert/etcd
source ~/ansible-core/bin/activate
ansible-inventory -i ../../etcd/inventory.ini --graph
```

## Run the Playbook

Use the existing etcd inventory explicitly:

```bash
cd /root/ansible-playbook/cert/etcd
source ~/ansible-core/bin/activate
ansible-playbook --syntax-check \
  -i ../../etcd/inventory.ini \
  issue_etcd_certificates.yml

ansible-playbook \
  -i ../../etcd/inventory.ini \
  issue_etcd_certificates.yml
```

The playbook performs these steps:

1. Generates one local etcd CA.
2. Generates one certificate and private key for each etcd node.
3. Installs the CA, certificate, and key under `/etc/etcd/pki`.
4. Migrates persisted HTTP peer URLs to HTTPS when the existing cluster is healthy.
5. Replaces the etcd systemd unit with a mutual TLS configuration.
6. Enables and restarts etcd.

The CA private key remains on the control machine and is not copied to the servers.

## Verify TLS

Run the following command from an etcd node after deployment:

```bash
ETCDCTL_API=3 /usr/local/bin/etcdctl \
  --endpoints=https://10.168.2.100:2379,https://10.168.2.101:2379,https://10.168.2.102:2379 \
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
source ~/ansible-core/bin/activate && ansible etcd01 -i ../../etcd/inventory.ini -b -m ansible.builtin.shell -a 'ETCDCTL_API=3 /usr/local/bin/etcdctl --endpoints=https://10.168.2.100:2379,https://10.168.2.101:2379,https://10.168.2.102:2379 --cacert=/etc/etcd/pki/ca.pem --cert=/etc/etcd/pki/server.pem --key=/etc/etcd/pki/server-key.pem endpoint status -w table'
## Important Warning

Do not run this playbook against a production etcd cluster without a maintenance plan. It changes client and peer endpoints from HTTP to HTTPS and restarts etcd. Keep a backup of the existing etcd data and configuration before deployment.

The generated CA private key is required to issue future certificates. Protect the `generated/` directory and back it up securely.
