# Ansible Ad Hoc Commands

Common Ansible ad hoc commands for this repository. Run them from `/root/ansible-playbook`.

## Activate Ansible

```bash
# Activate the Ansible virtual environment.
source ~/ansible-core/bin/activate

# Confirm the active Ansible version.
ansible --version
```

## Inventory and Connectivity

```bash
# List all hosts in the HAProxy and Keepalived inventory.
ansible-inventory -i keepalive_haproxy/inventory.ini --graph

# List all hosts in the etcd inventory.
ansible-inventory -i etcd/inventory.ini --graph

# Test SSH connectivity and Python availability.
ansible all -i etcd/inventory.ini -m ansible.builtin.ping

# Test only the HAProxy and Keepalived hosts.
ansible keepalive_haproxy -i keepalive_haproxy/inventory.ini -m ansible.builtin.ping

# Run a command on one host only.
ansible etcd01 -i etcd/inventory.ini -m ansible.builtin.ping
```

## Host Information

```bash
# Dispaly all groups
 ansible localhost -m debug -a 'var=groups'
# Display gathered facts for all etcd hosts.
ansible etcd -i etcd/inventory.ini -m ansible.builtin.setup

# Show the operating system and kernel version.
ansible all -i etcd/inventory.ini -m ansible.builtin.shell \
  -a 'cat /etc/os-release && uname -r'

# Show disk usage on all hosts.
ansible all -i etcd/inventory.ini -m ansible.builtin.command \
  -a 'df -h'

# Check memory usage.
ansible all -i etcd/inventory.ini -m ansible.builtin.command \
  -a 'free -h'
# Check network interface
ansible all -m setup -a 'gather_subset=network filter=ansible_*'
```

## Package Management

```bash
# Check whether etcd is installed.
ansible etcd -i etcd/inventory.ini -b -m ansible.builtin.package \
  -a 'name=etcd state=present' --check

# Install a package on all HAProxy hosts.
ansible keepalive_haproxy -i keepalive_haproxy/inventory.ini -b \
  -m ansible.builtin.apt -a 'name=curl state=present update_cache=yes'

# Remove a package from all target hosts.
ansible etcd -i etcd/inventory.ini -b -m ansible.builtin.apt \
  -a 'name=example-package state=absent'
```

## Service Management

```bash
# Check the status of etcd.
ansible etcd -i etcd/inventory.ini -b -m ansible.builtin.systemd \
  -a 'name=etcd state=started enabled=yes'

# Restart etcd on one host.
ansible etcd01 -i etcd/inventory.ini -b -m ansible.builtin.systemd \
  -a 'name=etcd state=restarted'

# Check HAProxy and Keepalived service status.
ansible keepalive_haproxy -i keepalive_haproxy/inventory.ini -b \
  -m ansible.builtin.systemd -a 'name=haproxy state=started'
ansible keepalive_haproxy -i keepalive_haproxy/inventory.ini -b \
  -m ansible.builtin.systemd -a 'name=keepalived state=started'

# View recent service logs.
ansible etcd -i etcd/inventory.ini -b -m ansible.builtin.command \
  -a 'journalctl -u etcd -n 50 --no-pager'
```

## Files and Directories

```bash
# Check whether a configuration file exists and display its permissions.
ansible etcd -i etcd/inventory.ini -b -m ansible.builtin.stat \
  -a 'path=/etc/systemd/system/etcd.service'

# Display the deployed etcd systemd unit.
ansible etcd -i etcd/inventory.ini -b -m ansible.builtin.command \
  -a 'cat /etc/systemd/system/etcd.service'

# Create a directory on all etcd hosts.
ansible etcd -i etcd/inventory.ini -b -m ansible.builtin.file \
  -a 'path=/var/lib/example state=directory owner=root group=root mode=0755'

# Copy a local file to all etcd hosts.
ansible etcd -i etcd/inventory.ini -b -m ansible.builtin.copy \
  -a 'src=./local-file.conf dest=/etc/local-file.conf owner=root group=root mode=0644'
```

## etcd Checks

```bash
# Check the health of all etcd endpoints.
ansible etcd01 -i etcd/inventory.ini -b -m ansible.builtin.shell \
  -a 'ETCDCTL_API=3 /usr/local/bin/etcdctl --endpoints=http://10.168.2.100:2379,http://10.168.2.101:2379,http://10.168.2.102:2379 endpoint health'

# List etcd cluster members.
ansible etcd01 -i etcd/inventory.ini -b -m ansible.builtin.shell \
  -a 'ETCDCTL_API=3 /usr/local/bin/etcdctl --endpoints=http://10.168.2.100:2379,http://10.168.2.101:2379,http://10.168.2.102:2379 member list'
```

## Safe Checks

```bash
# Preview a command without changing remote hosts when the module supports check mode.
ansible all -i etcd/inventory.ini -b -m ansible.builtin.apt \
  -a 'name=curl state=present' --check

# Run a command with verbose output for troubleshooting.
ansible etcd -i etcd/inventory.ini -m ansible.builtin.ping -vvv
```

Avoid putting passwords, tokens, or private keys directly in ad hoc command history. Do not delete `/var/lib/etcd` unless destroying the existing etcd data is intentional.
