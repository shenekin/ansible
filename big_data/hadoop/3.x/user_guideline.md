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

