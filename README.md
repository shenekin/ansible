# Ansible Infrastructure Playbooks

This repository contains Ansible playbooks for:

- A three-node HAProxy and Keepalived high-availability cluster.
- A three-node etcd 3.5 cluster.

The current inventories use the following hosts:

| Host | Address |
|---|---|
| Node 1 | 10.168.2.100 |
| Node 2 | 10.168.2.101 |
| Node 3 | 10.168.2.102 |

## Requirements

- Linux control machine with SSH access to all target hosts.
- Ansible environment at `~/ansible-core`.
- Remote access as `root`, or an account with passwordless sudo.
- Debian/Ubuntu targets for the package installation tasks.

Activate Ansible before running commands:

```bash
source ~/ansible-core/bin/activate
```

Review the relevant inventory before deployment, especially host addresses, network interface names, VIPs, and passwords.

## HAProxy and Keepalived

Files:

- `keepalive_haproxy/inventory.ini`
- `keepalive_haproxy/deploy_haproxy_keepalived.yml`
- `keepalive_haproxy/templates/`

The current Keepalived VIP is `10.168.2.188`. The nodes use priorities `100`, `90`, and `80`. HAProxy listens on port `80`, and its statistics page uses port `8404`.

Validate and deploy:

```bash
cd /root/ansible-playbook/keepalive_haproxy
source ~/ansible-core/bin/activate
ansible-inventory -i inventory.ini --graph
ansible-playbook --syntax-check -i inventory.ini deploy_haproxy_keepalived.yml
ansible-playbook -i inventory.ini deploy_haproxy_keepalived.yml
```

## etcd 3.5

Files:

- `etcd/inventory.ini`
- `etcd/deploy_etcd.yml`
- `etcd/etcd.service.j2`

The etcd cluster uses version `3.5.16`, client port `2379`, and peer port `2380`.

Validate and deploy:

```bash
cd /root/ansible-playbook/etcd
source ~/ansible-core/bin/activate
ansible-inventory -i inventory.ini --graph
ansible-playbook --syntax-check -i inventory.ini deploy_etcd.yml
ansible-playbook -i inventory.ini deploy_etcd.yml
```

Check etcd health after deployment:

```bash
ETCDCTL_API=3 /usr/local/bin/etcdctl \
	--endpoints=http://10.168.2.100:2379,http://10.168.2.101:2379,http://10.168.2.102:2379 \
	endpoint health
```

Do not run the etcd playbook with `etcd_initial_cluster_state=new` against an existing cluster unless the data and migration procedure have been verified. Removing `/var/lib/etcd` destroys the local etcd data.