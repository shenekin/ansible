## Hadoop User and Group Configuration

The configuration uses two distinct identities:

- **Ansible connection user:** `ubuntu`, configured in `ansible.cfg:3`. The installation playbook enables privilege escalation, so its tasks run as root unless another user is specified.
- **Hadoop service user and group:** `hadoop:hadoop`, configured in `hadoop_configuration.yaml:6-7`. The playbook creates them as system accounts, sets the user’s shell to `/usr/sbin/nologin`, and assigns Hadoop’s directories to that account. The systemd units also run the Hadoop services as `hadoop:hadoop`, for example `namenode.service.j2:8-9`.

This is a sound general pattern: Ansible connects as `ubuntu`, while Hadoop runs under a dedicated, non-interactive service account.

Two points are worth checking:

1. **NameNode formatting runs as root.** The formatting task inherits `become: true` and does not specify `become_user: hadoop` (`install-hadoop.yml:208-210`). Hadoop services later run as `hadoop`, so check that the NameNode metadata directory is owned by `hadoop:hadoop` after deployment. Running the format task as `hadoop` would keep the identities consistent.
2. **Hadoop CLI commands run as whoever invokes them.** The environment playbook makes the commands available system-wide, but it does not switch the caller to `hadoop` (`hadoop_configration.yml:25-42`). For example, running `hdfs` from an `ubuntu` shell means the client uses the `ubuntu` identity. Since HDFS permissions are enabled, some administrative commands may need to be run as the appropriate Hadoop identity or HDFS superuser; ordinary file operations should use the intended user’s HDFS permissions.

One additional consideration: if a `hadoop` account already exists on a host with supplementary groups, the playbook does not explicitly remove or constrain those groups. For newly created accounts, this is generally not an issue.

## Operating Hadoop as `ubuntu`

### Recommended account model

Use `ubuntu` for SSH access, Ansible, and normal Hadoop client commands. Keep the NameNode, DataNode, ResourceManager, and NodeManager daemons running as `hadoop:hadoop`. The `hadoop` account is intentionally a system account with `/usr/sbin/nologin`; it does not need an interactive login shell for `sudo -u hadoop` to run a command.

This distinction protects the existing HDFS metadata and block files. The configured NameNode and DataNode paths are `/data/hadoop/hdfs/namenode` and `/data/hadoop/hdfs/datanode`, and Hadoop directories are created for the `hadoop` account. Do not change `hadoop.owner` to `ubuntu` or recursively change ownership of `/data/hadoop` to make CLI commands work.

### 1. Confirm connectivity and inventory

Run the following commands from this Hadoop 3.x directory on the control node. The active inventory has `hadoop01` at `10.168.2.54`, and `hadoop02` and `hadoop03` as worker hosts. Check the inventory and SSH access first:

```bash
ansible-inventory -i inventory.ini --graph
ansible hadoop_cluster -i inventory.ini -m ansible.builtin.ping
```

The configured Ansible connection user is `ubuntu`. Resolve any `UNREACHABLE` result before running playbooks.

### 2. Install the system-wide Hadoop environment

The existing `hadoop_configration.yml` playbook configures `/etc/profile.d/hadoop.sh` and command links in `/usr/local/bin`; it does not change Hadoop's service user or data ownership. Preview and validate the playbook, then apply it:

```bash
ansible-playbook -i inventory.ini hadoop_configration.yml --syntax-check
ansible-playbook -i inventory.ini hadoop_configration.yml --check --diff
ansible-playbook -i inventory.ini hadoop_configration.yml
```

Open a new SSH session as `ubuntu`, or load the profile in the current shell:

```bash
source /etc/profile.d/hadoop.sh
whoami
echo "$JAVA_HOME"
echo "$HADOOP_HOME"
echo "$HADOOP_CONF_DIR"
command -v hadoop
command -v hdfs
command -v yarn
```

Expected values include `ubuntu` for `whoami`, `/opt/hadoop/current` for `HADOOP_HOME`, and `/opt/hadoop/current/etc/hadoop` for `HADOOP_CONF_DIR`. The configured Java path is `/usr/lib/jvm/java-11-openjdk-amd64`.

### 3. Check that the Hadoop services are healthy

Run these checks on the relevant hosts. `systemctl` administration is performed through `sudo`; the services themselves continue to run as `hadoop`:

```bash
sudo systemctl status hadoop-namenode
sudo systemctl status hadoop-datanode
sudo systemctl status hadoop-resource-manager
sudo systemctl status hadoop-node-manager
jps
```

The NameNode and ResourceManager are configured on `hadoop01`; DataNodes and NodeManagers are configured on all three hosts. Use `sudo systemctl start`, `stop`, or `restart` with the applicable unit name when service control is required. Do not invoke Hadoop's daemon scripts directly as `ubuntu` to replace the systemd-managed services.

### 4. Initialize Ubuntu's HDFS home directory

HDFS authenticates client requests as the operating-system identity that runs the client. A command run by `ubuntu` is an HDFS request from `ubuntu`; the operating-system `sudo` permission does not automatically make that request an HDFS superuser. In this cluster the NameNode daemon runs as `hadoop`, so use `sudo -u hadoop` for the one-time setup of Ubuntu's HDFS home directory.

Run the following on a host with access to the configured HDFS NameNode, normally `hadoop01`, after the NameNode is running:

```bash
sudo -u hadoop env \
	JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64 \
	HADOOP_HOME=/opt/hadoop/current \
	HADOOP_CONF_DIR=/opt/hadoop/current/etc/hadoop \
	/opt/hadoop/current/bin/hdfs dfs -mkdir -p /user/ubuntu

sudo -u hadoop env \
	JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64 \
	HADOOP_HOME=/opt/hadoop/current \
	HADOOP_CONF_DIR=/opt/hadoop/current/etc/hadoop \
	/opt/hadoop/current/bin/hdfs dfs -chown ubuntu /user/ubuntu
```

If the directory already contains data, inspect it before changing its owner. Do not recursively change ownership of HDFS storage on the local filesystem; `/user/ubuntu` is an HDFS path, not `/data/hadoop`.

### 5. Use HDFS and YARN as Ubuntu

After the HDFS home directory exists, run ordinary client commands directly from the `ubuntu` shell. HDFS permissions remain enabled, so Ubuntu can read, write, or remove only paths allowed by HDFS ownership, permissions, and ACLs.

```bash
hdfs dfs -ls /
hdfs dfs -mkdir -p /user/ubuntu/input
hdfs dfs -put ./input.txt /user/ubuntu/input/
hdfs dfs -ls /user/ubuntu/input
yarn application -list
```

Submit MapReduce jobs as `ubuntu` using the cluster's configured YARN framework, for example:

```bash
hadoop jar /opt/hadoop/current/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.3.6.jar \
	pi 2 10
```

Use HDFS paths owned by `ubuntu` for job input and output, or have an HDFS administrator grant the required access. Do not use `sudo` for routine jobs unless the job is intentionally meant to run as another operating-system user.

### 6. Run HDFS superuser commands when needed

Some HDFS administrative operations require the NameNode's HDFS superuser identity. Run those commands as `hadoop` with `sudo -u`, rather than changing the service account or pretending to be `hadoop` from the Ubuntu shell. For example:

```bash
sudo -u hadoop env \
	JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64 \
	HADOOP_HOME=/opt/hadoop/current \
	HADOOP_CONF_DIR=/opt/hadoop/current/etc/hadoop \
	/opt/hadoop/current/bin/hdfs dfsadmin -report
```

Use the same `sudo -u hadoop env ...` prefix for other HDFS superuser commands, such as `hdfs dfsadmin` operations. Do not set `HADOOP_USER_NAME=hadoop` as an Ubuntu user to bypass HDFS authorization; in simple authentication that is not a secure privilege boundary.

## Open a Hadoop Service-User Shell

The `hadoop` account is configured with `/usr/sbin/nologin`, so direct SSH login and `sudo -iu hadoop` are not the intended access methods. You can still start a shell as this account from your existing `ubuntu` SSH session. This does not change the account's login shell, password, or SSH access, and it keeps the Hadoop systemd services running under their configured `hadoop:hadoop` identity.

### Interactive shell

First connect to a cluster node as `ubuntu`, then start Bash explicitly as `hadoop`:

```bash
ssh -i ~/.ssh/id_ed25519_pve_host ubuntu@10.168.2.54
sudo -u hadoop -H /bin/bash
source /etc/profile.d/hadoop.sh
```

Confirm the effective identity and environment:

```bash
whoami
id
echo "$HOME"
echo "$JAVA_HOME"
echo "$HADOOP_HOME"
command -v hadoop
command -v hdfs
command -v yarn
```

Expected values include `hadoop` for `whoami`, `/opt/hadoop` for `HOME`, `/usr/lib/jvm/java-11-openjdk-amd64` for `JAVA_HOME`, and `/opt/hadoop/current` for `HADOOP_HOME`. Run the following as `hadoop` to verify HDFS administrative access and YARN connectivity:

```bash
hdfs dfsadmin -report
hdfs dfs -ls /
yarn application -list
```

Use `exit` to leave the `hadoop` shell and return to the `ubuntu` shell. The interactive Bash was started explicitly; this does not enable SSH login for the service account.

### Run a single command as Hadoop

To run one Hadoop command without opening an interactive shell, explicitly load the system-wide environment:

```bash
sudo -u hadoop -H /bin/bash -c \
	'source /etc/profile.d/hadoop.sh && hdfs dfsadmin -report'
```

For example, to run an HDFS command or submit a YARN job as the `hadoop` operating-system user:

```bash
sudo -u hadoop -H /bin/bash -c \
	'source /etc/profile.d/hadoop.sh && hdfs dfs -ls /'

sudo -u hadoop -H /bin/bash -c \
	'source /etc/profile.d/hadoop.sh && yarn application -list'
```

Commands run this way are authenticated as `hadoop` by Hadoop's simple authentication configuration. HDFS superuser actions should be limited to administrative work; routine jobs should normally run as their intended user so HDFS permissions and YARN application ownership remain meaningful.

### What still requires Ubuntu sudo

The service account owns Hadoop processes and data, but systemd service management should still be done from the `ubuntu` account with its authorized `sudo` access:

```bash
sudo systemctl status hadoop-namenode hadoop-datanode \
	hadoop-resource-manager hadoop-node-manager
sudo systemctl restart hadoop-datanode
```

Do not run Hadoop daemon start/stop scripts manually from the service-user shell to replace systemd. The unit files are the source of truth for daemon lifecycle and environment.

### Account security and current group membership

Do not enable direct SSH login or set an interactive password for `hadoop` just to use Hadoop commands. Anyone who can authenticate as `hadoop` can act as the HDFS superuser in this cluster. The supported access path is SSH as `ubuntu`, followed by `sudo -u hadoop` for the required command or shell.

The live account check on Nov 12, 2026 found that `hadoop` belongs to the operating-system `sudo` group on `hadoop01` and `hadoop03`, but not on `hadoop02`. Hadoop does not need this group to run its services or HDFS commands. Review that membership and, if it was not explicitly intended, remove it on the affected hosts using an authorized administrator account:

```bash
sudo gpasswd -d hadoop sudo
```

This changes only local Linux sudo-group membership; it is not required for `sudo -u hadoop` from `ubuntu`, and it does not grant or revoke HDFS permissions. Verify with `id hadoop` on each host after any approved group change.

### Troubleshooting

- **`hadoop`, `hdfs`, or `yarn` is not found:** open a new login session or run `source /etc/profile.d/hadoop.sh`; confirm the symlinks in `/usr/local/bin` and the paths in `vars/hadoop_configuration.yaml` and `vars/hadoop_env_configuration.yml`.
- **Java cannot be found:** verify that `/usr/lib/jvm/java-11-openjdk-amd64/bin/java` exists and that `JAVA_HOME` matches the configured Java installation.
- **HDFS reports `Permission denied`:** confirm the client identity with `whoami`, check HDFS ownership and permissions with `hdfs dfs -ls`, and use an HDFS administrator to grant access. Do not fix HDFS authorization by changing local `/data/hadoop` ownership.
- **HDFS reports connection refused or cannot connect to `hadoop01:9000`:** check NameNode health, DNS/host resolution, and network access to the configured `fs.defaultFS` endpoint.
- **A service fails after a manual change:** inspect `sudo systemctl status <unit>` and `sudo journalctl -u <unit>`. Confirm that the unit still specifies `User=hadoop` and `Group=hadoop` and that the Hadoop data and runtime directories remain accessible to that account.
- **NameNode metadata ownership is uncertain:** inspect `/data/hadoop/hdfs/namenode` and verify it is owned by `hadoop:hadoop`. The install playbook formats a new NameNode only when metadata is absent, then sets the metadata directory ownership. Never run `hdfs namenode -format` against an existing cluster as a troubleshooting step.

### Changes to avoid

- Do not change `hadoop.owner` or `hadoop.group` to `ubuntu` for an existing cluster.
- Do not change the `User=` or `Group=` settings in the four systemd service templates to `ubuntu`.
- Do not run `chown -R ubuntu` on `/data/hadoop`, `/opt/hadoop`, `/var/log/hadoop`, or `/var/run/hadoop`.
- Do not reformat an initialized NameNode or delete NameNode/DataNode directories.
- Do not enable an interactive shell for the `hadoop` system account just to run its commands; `sudo -u hadoop` is sufficient.

The recommended result is that `ubuntu` can run authorized HDFS and YARN client operations and manage systemd through `sudo`, while Hadoop daemons continue to run as `hadoop:hadoop` and retain their existing filesystem ownership.

## Grant Ubuntu HDFS Superuser Permissions

This cluster uses Hadoop simple authentication, with `dfs.permissions.enabled=true`. The NameNode uses the Linux group configured by `dfs.permissions.superusergroup` as its HDFS superuser group. The active value is Hadoop's default, `supergroup`. To let `ubuntu` run HDFS administrative commands under its own identity, make that Linux group available on every Hadoop node and add `ubuntu` to it. This does not change the Hadoop service user, NameNode metadata, DataNode storage, or systemd units.

The Ansible configuration for this access is kept separate from the main install playbook:

- `configure-hadoop-admin-user.yml` creates the group, adds `ubuntu` as a supplementary member on all cluster hosts, and refreshes the NameNode's user-to-group mapping.
- `vars/hadoop_admin_user.yml` defines the account, group, group ID, and Hadoop runtime paths.

The configured group ID is `987`; it was checked to be unused on all three current nodes. Use one consistent GID on every node. Before applying this configuration to a different cluster, verify that the chosen GID is free on every host.

### Apply the configuration

Run these commands from the Hadoop 3.x project directory on the control node, with the Ansible environment activated:

```bash
source ~/.pyenv/versions/ansible-2.21-env/bin/activate
cd /Users/ekin/Documents/Development_Codes/ansible/big_data/hadoop/3.x
ansible-playbook -i inventory.ini configure-hadoop-admin-user.yml --syntax-check
ansible-playbook -i inventory.ini configure-hadoop-admin-user.yml
```

The playbook is idempotent and can be run again safely. It adds `supergroup` as a supplementary group; it does not replace the account's existing groups. Its NameNode task refreshes the group cache without restarting Hadoop services. In Ansible check mode, creation of a group is only simulated, so a following user task may report that the group does not exist. Use the syntax check and a normal playbook run for this dependent group-and-user workflow.

After the playbook completes, start a new SSH session as `ubuntu`. A session opened before the group change may not have the new supplementary group in its process credentials.

### Verify HDFS administrative permissions as Ubuntu

Run these commands directly as `ubuntu`, without `sudo -u hadoop`:

```bash
source /etc/profile.d/hadoop.sh
whoami
id
hdfs groups ubuntu
hdfs dfsadmin -report
```

Expected results include `ubuntu` from `whoami`, `supergroup` in the output of `id` and `hdfs groups ubuntu`, and a live DataNode report from `hdfs dfsadmin -report`. The current cluster has three live DataNodes.

An optional write check can create and remove a temporary HDFS directory. Use an existing HDFS parent such as `/user`; do not assume `/tmp` exists in HDFS:

```bash
test_path=/user/.ubuntu-admin-check
hdfs dfs -mkdir "$test_path"
hdfs dfs -test -d "$test_path"
hdfs dfs -rmdir "$test_path"
```

### YARN and MapReduce permissions

The current Hadoop configuration reports `yarn.admin.acl=*`, `yarn.acl.enable=false`, and `mapreduce.cluster.acls.enabled=false`. Under these current settings, no separate YARN or MapReduce ACL change is needed for `ubuntu` to use the available commands. For example:

```bash
yarn application -list
yarn node -list
```

An empty application list is normal when no applications are running. If YARN ACLs are enabled or customized later, review `yarn.admin.acl`, `yarn.acl.enable`, and the MapReduce ACL properties separately; membership in HDFS `supergroup` does not automatically grant privileges in every Hadoop subsystem.

### Permission scope and security

Membership in `supergroup` gives `ubuntu` HDFS superuser privileges. This is equivalent to the Hadoop service identity for HDFS authorization, but it does not make `ubuntu` the Linux `hadoop` account, does not grant additional Linux filesystem ownership, and does not make it operating-system root. Use `sudo` separately for systemd or other operating-system administration.

Treat the Ubuntu SSH credentials as highly privileged: a compromised `ubuntu` account can perform HDFS administrative operations. Do not add general-purpose users to `supergroup`. Keep the daemons running as `hadoop:hadoop`, and do not change `/data/hadoop` ownership or reformat the NameNode to grant user permissions.

