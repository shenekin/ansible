## Hadoop User and Group Configuration

The configuration uses two distinct identities:

- **Ansible connection user:** `ubuntu`, configured in `ansible.cfg:3`. The installation playbook enables privilege escalation, so its tasks run as root unless another user is specified.
- **Hadoop service user and group:** `hadoop:hadoop`, configured in `hadoop_configuration.yaml:6-7`. The playbook creates them as system accounts, sets the user’s shell to `/usr/sbin/nologin`, and assigns Hadoop’s directories to that account. The systemd units also run the Hadoop services as `hadoop:hadoop`, for example `namenode.service.j2:8-9`.

This is a sound general pattern: Ansible connects as `ubuntu`, while Hadoop runs under a dedicated, non-interactive service account.

Two points are worth checking:

1. **NameNode formatting runs as root.** The formatting task inherits `become: true` and does not specify `become_user: hadoop` (`install-hadoop.yml:208-210`). Hadoop services later run as `hadoop`, so check that the NameNode metadata directory is owned by `hadoop:hadoop` after deployment. Running the format task as `hadoop` would keep the identities consistent.
2. **Hadoop CLI commands run as whoever invokes them.** The environment playbook makes the commands available system-wide, but it does not switch the caller to `hadoop` (`hadoop_configration.yml:25-42`). For example, running `hdfs` from an `ubuntu` shell means the client uses the `ubuntu` identity. Since HDFS permissions are enabled, some administrative commands may need to be run as the appropriate Hadoop identity or HDFS superuser; ordinary file operations should use the intended user’s HDFS permissions.

One additional consideration: if a `hadoop` account already exists on a host with supplementary groups, the playbook does not explicitly remove or constrain those groups. For newly created accounts, this is generally not an issue.

