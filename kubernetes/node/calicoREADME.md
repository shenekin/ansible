# Calico deployment guide

This directory contains a separate Calico deployment playbook and its configuration file so the manifest and image configuration remain independent.

## File layout

- `calico-deploy.yml`: deploys the Calico manifest to the cluster.
- `calico-config.yml`: stores image registry, image tags, and cluster pod CIDR values.
- `calico.yaml`: the original upstream-style Calico manifest used as the base manifest.

## Important behavior

- The deployment does not modify the existing node installation playbooks.
- The image values can be changed in `calico-config.yml` without editing the manifest.
- The manifest is copied to a temporary file, patched in place, and then applied with `kubectl`.

## How to deploy Calico

From the node directory:

```bash
cd /root/ansible-playbook/kubernetes/node
source ~/ansible-core/bin/activate
ansible-playbook -i inventory.ini calico-deploy.yml
```

## How to change the image source

Edit the `calico-config.yml` file and change values such as:

```yaml
calico_image_registry: docker.m.daocloud.io
calico_image_repository: calico
calico_version: v3.26.5
```

This will automatically update the image references in the rendered Calico manifest.

## How to change the cluster pod CIDR

Edit the following value in `calico-config.yml`:

```yaml
calico_ipv4pool_cidr: "192.168.0.0/16"
```

## Notes

- The manifest is applied using the cluster admin kubeconfig at `/etc/kubernetes/admin.conf`.
- This playbook is designed to be safe and separate from the existing node installation flow.
- The original `calico.yaml` file remains the source manifest and is not changed by the playbook.
