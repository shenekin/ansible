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
