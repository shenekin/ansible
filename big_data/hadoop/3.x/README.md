# Hadoop 3.x Ansible Deployment

This directory deploys an Apache Hadoop 3.x cluster with Ansible.

## Configuration

All deployment and Hadoop settings are maintained in:

```text
vars/hadoop_configuration.yaml
```

Host addresses and Ansible groups are maintained in:

```text
inventory.ini
```

All Hadoop XML files, environment files, worker files, and systemd units are stored as templates in:

```text
templates/
```

## Run the Playbook

From this directory, run:

```bash
ansible-playbook -i inventory.ini install-hadoop.yml
```

Preview changes before applying them:

```bash
ansible-playbook -i inventory.ini install-hadoop.yml --check --diff
```

Validate inventory and syntax:

```bash
ansible-inventory -i inventory.ini --graph
ansible-playbook -i inventory.ini install-hadoop.yml --syntax-check
```

The installation playbook configures `/run/hadoop` to be recreated at boot with
the Hadoop service account as its owner. This is required because `/var/run` is
temporary and is cleared during a reboot.

## Configure the Hadoop Shell Environment

After Hadoop and Java are installed on the cluster nodes, configure the Hadoop
environment for command-line use on every host in the `hadoop_cluster` inventory
group. The playbook reads the installation paths from
`vars/hadoop_configuration.yaml` and the environment settings from
`vars/hadoop_env_configuration.yml`. It adds a dedicated
`/etc/profile.d/hadoop.sh` file and command links in `/usr/local/bin` without
changing the Hadoop installation or its service configuration. The command
links make Hadoop commands available even in shells that have not loaded the
profile script yet.

Preview and apply the configuration from this directory:

```bash
ansible-playbook -i inventory.ini hadoop_configration.yml --check --diff
ansible-playbook -i inventory.ini hadoop_configration.yml
```

Load the environment settings in the current shell, or start a new login
session:

```bash
source /etc/profile.d/hadoop.sh
```

## Manually Verify Hadoop

Run the following checks on each Hadoop node after applying the environment
playbook. `source` updates the current shell; a new login session loads the
profile automatically.

```bash
source /etc/profile.d/hadoop.sh
whoami
echo "$HOME"
echo "$JAVA_HOME"
echo "$HADOOP_HOME"
command -v java
command -v hadoop
command -v hdfs
command -v yarn
java -version
hadoop version
hdfs version
yarn version
hadoop
```

`command -v hadoop` should resolve to `/usr/local/bin/hadoop` or the Hadoop
installation's `bin` directory. `hadoop` without arguments should print its
usage information. To verify HDFS access as well, when the cluster services are
running, run:

```bash
hdfs dfs -ls /
```

Validate the playbook syntax before running it:

```bash
ansible-playbook -i inventory.ini hadoop_configration.yml --syntax-check
```
