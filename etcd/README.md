# etcd 3.5 Ansible Deployment

This directory contains an Ansible playbook for deploying a three-node etcd 3.5 cluster.

## Cluster Layout

| Node | Address | Client port | Peer port |
|---|---|---:|---:|
| etcd01 | 10.168.2.100 | 2379 | 2380 |
| etcd02 | 10.168.2.101 | 2379 | 2380 |
| etcd03 | 10.168.2.102 | 2379 | 2380 |

The current configuration uses etcd `3.5.16` and the `linux-amd64` release archive.

## Files

- `inventory.ini`: Hosts, addresses, names, ports, and cluster settings.
- `deploy_etcd.yml`: Main deployment playbook.
- `etcd.service.j2`: systemd service template.

## Prerequisites

- Ansible installed in `~/ansible-core`.
- SSH access to all three nodes.
- The remote account must be able to use `sudo`. The supplied inventory uses `root`.
- Ubuntu or another Debian-based system with `apt`, unless package installation is adapted.
- Network connectivity between all nodes on TCP ports `2379` and `2380`.

Activate the Ansible environment before running commands:

```bash
source ~/ansible-core/bin/activate
```

## Configure the Inventory

Review `inventory.ini` before deployment. Update the following values when necessary:

- `ansible_user`
- `etcd_version`
- `etcd_data_dir`
- `etcd_initial_cluster`
- `etcd_client_port`
- `etcd_peer_port`

The `etcd_name` value for each host must match the corresponding name in `etcd_initial_cluster`.

## Validate the Configuration

Run these commands from this directory:

```bash
source ~/ansible-core/bin/activate
ansible-inventory -i inventory.ini --graph
ansible-playbook --syntax-check -i inventory.ini deploy_etcd.yml
```

## Deploy the Cluster

Run the playbook against all three nodes:

```bash
source ~/ansible-core/bin/activate
ansible-playbook -i inventory.ini deploy_etcd.yml
```

The playbook creates the `etcd` system user, downloads the official release, installs the binaries, creates the data directory, installs the systemd unit, and starts etcd.

## Verify the Cluster

Run these commands on any node after deployment:

```bash
sudo systemctl status etcd
sudo ETCDCTL_API=3 /usr/local/bin/etcdctl \
  --endpoints=http://10.168.2.100:2379,http://10.168.2.101:2379,http://10.168.2.102:2379 \
  endpoint health
```

Check the member list:

```bash
sudo ETCDCTL_API=3 /usr/local/bin/etcdctl \
  --endpoints=http://10.168.2.100:2379,http://10.168.2.101:2379,http://10.168.2.102:2379 \
  member list
```

## Important Data Warning

This playbook creates a new cluster with `etcd_initial_cluster_state=new`. Do not use it against an existing etcd data directory unless you have confirmed the intended migration procedure.

To rebuild a test cluster, stop etcd and remove the data directory on all members before running the playbook again:

```bash
sudo systemctl stop etcd
sudo rm -rf /var/lib/etcd/*
```

Only remove the data directory when destroying the existing cluster is intentional.
