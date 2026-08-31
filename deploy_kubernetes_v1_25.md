# Kubernetes v1.25 Deployment Guide

This document describes the deployment logic and execution order for the Kubernetes cluster defined in this repository. It is written based on the actual Ansible playbooks, inventory files, and generated certificate assets currently present in this workspace.

The deployment model in this repository is a Kubernetes v1.25 cluster with:

- A three-node HAProxy + Keepalived load-balancing layer
- A three-node etcd 3.5 cluster
- A three-node Kubernetes control-plane cluster
- A set of worker nodes for kubelet and kube-proxy
- Containerd as the runtime
- TLS certificate generation for both etcd and Kubernetes components

The repository is organized around a clear sequence: infrastructure load balancer first, then etcd, then certificate prerequisites, then container runtime, then Kubernetes control plane, and finally node installation.

---

## 1. Deployment Philosophy and Execution Order

The logic of this repository is intentional and linear. The playbooks assume that dependencies are created in sequence, because each later stage depends on the outputs of the earlier one.

The recommended order is:

1. Review infrastructure networking and host mapping.
2. Deploy the HAProxy + Keepalived service layer.
3. Deploy the etcd cluster.
4. Generate and validate Kubernetes control-plane certificates.
5. Install and configure containerd on all runtime hosts.
6. Deploy the Kubernetes control plane on the master nodes.
7. Install kubelet and kube-proxy on the worker nodes.
8. Validate cluster readiness.

This is not a single monolithic playbook. It is a layered deployment model where each module has a distinct responsibility.

---

## 2. Current Cluster Topology in This Repository

The repository inventory files define the following topology.

### 2.1 Load Balancing Layer

From `keepalive_haproxy/inventory.ini`:

- lb01: 10.168.2.100
- lb02: 10.168.2.101
- lb03: 10.168.2.102

VIP:

- 10.168.2.188

The cluster uses:

- VRRP router ID: 51
- VRRP interface: `ens18`
- HAProxy frontend port: 80
- HAProxy stats port: 8404

This VIP is the external API address used by Kubernetes clients and kubeconfigs.

### 2.2 etcd Cluster

From `etcd/inventory.ini`:

- etcd01: 10.168.2.108
- etcd02: 10.168.2.109
- etcd03: 10.168.2.110

Version:

- etcd 3.5.16

Ports:

- client: 2379
- peer: 2380

The cluster is configured as a new cluster using `etcd_initial_cluster_state=new`.

### 2.3 Kubernetes Control Plane

From `kubernetes/master/inventory.ini`:

- k8s-master-01: 10.168.2.100
- k8s-master-02: 10.168.2.101
- k8s-master-03: 10.168.2.102

The Kubernetes API server address is configured as:

- 10.168.2.188

This is the client-facing address for the cluster and should match the load-balancer VIP.

The Kubernetes version used here is:

- v1.25.16

### 2.4 Kubernetes Nodes

From `kubernetes/node/inventory.ini`:

- k8s-master-01: 10.168.2.100
- k8s-master-02: 10.168.2.101
- k8s-master-03: 10.168.2.102
- k8s-node-01: 10.168.2.103
- k8s-node-02: 10.168.2.104
- k8s-node-03: 10.168.2.105
- k8s-node-04: 10.168.2.106
- k8s-node-05: 10.168.2.107

The node playbook supports Debian/Ubuntu amd64 hosts and installs kubelet and kube-proxy.

### 2.5 Runtime Hosts

From `containers/containerd/inventory.ini`:

- containerd is configured on the same node set as the control plane and worker nodes.

The runtime is containerd 1.6.24 with runc 1.1.10 and crictl 1.25.0.

---

## 3. Repository Components and Their Purpose

### 3.1 `keepalive_haproxy/`

This directory deploys the Kubernetes API load-balancing layer.

Files:

- `keepalive_haproxy/deploy_haproxy_keepalived.yml`
- `keepalive_haproxy/inventory.ini`
- `keepalive_haproxy/templates/haproxy.cfg.j2`
- `keepalive_haproxy/templates/keepalived.conf.j2`

Purpose:

- Keepalived creates the VIP 10.168.2.188.
- HAProxy exposes the API endpoint and balances requests to the control-plane nodes.
- This layer provides high availability for the control plane.

Why this is first:

- Kubernetes clients, kubeconfigs, and the control-plane k8s service must be able to reach the API server through a stable address before the Kubernetes cluster is started.

### 3.2 `etcd/`

This directory deploys the etcd cluster.

Files:

- `etcd/deploy_etcd.yml`
- `etcd/inventory.ini`
- `etcd/etcd.service.j2`

Purpose:

- etcd stores the Kubernetes cluster data, including the API server state, cluster state, and distributed coordination data.
- It must exist and be healthy before the Kubernetes control plane can start successfully.

### 3.3 `cert/etcd/`

This directory is responsible for generating and issuing etcd TLS certificates.

Files:

- `cert/etcd/generate_etcd_certificates.yml`
- `cert/etcd/issue_etcd_certificates.yml`

Purpose:

- Generate CA and member certificates for the etcd cluster
- Establish secure client and peer communication
- Supply the certificates used by kube-apiserver when it connects to etcd over TLS

### 3.4 `cert/k8s/master/`

This directory is responsible for generating Kubernetes control-plane certificates.

Files:

- `cert/k8s/master/issue_control_plane_certificates.yml`
- `cert/k8s/master/generated/`

Purpose:

- Generate CA, front-proxy CA, API server certs, admin cert, controller-manager cert, scheduler cert, and service-account keypair
- Create the PKI structure required by the control plane

### 3.5 `containers/containerd/`

This directory installs containerd and the CRI tooling.

Files:

- `containers/containerd/deploy_containerd.yml`
- `containers/containerd/inventory.ini`

Purpose:

- Install containerd, runc, and crictl
- Configure the systemd cgroup mode for Kubernetes compatibility
- Make the container runtime available to kubelet on every node

### 3.6 `kubernetes/master/`

This directory is for the Kubernetes control plane.

Files:

- `kubernetes/master/deploy_control_plane.yml`
- `kubernetes/master/inventory.ini`
- `kubernetes/master/admission-config.yaml`
- `kubernetes/master/templates/*.j2`

Purpose:

- Download the Kubernetes v1.25 server binaries
- Install kube-apiserver, kube-controller-manager, kube-scheduler, and kubectl
- Configure TLS and kubeconfigs
- Start the control plane with the correct systemd units

### 3.7 `kubernetes/node/`

This directory installs the Kubernetes node agents.

Files:

- `kubernetes/node/deploy_node_install_uninstall.yml`
- `kubernetes/node/inventory.ini`
- `kubernetes/node/templates/*.j2`

Purpose:

- Download kubelet and kube-proxy
- Install node-specific certificates
- Render kubeconfig files
- Configure systemd units and start the service daemons

---

## 4. Prerequisites Before Deployment

Before running the playbooks, the environment must satisfy several prerequisites.

### 4.1 Control Machine Requirements

The control machine is the host that runs Ansible. The repository expects a Python/Ansible environment such as:

```bash
source ~/ansible-core/bin/activate
```

The README explicitly states that the environment is expected under `~/ansible-core`.

### 4.2 SSH Access

All target hosts must be reachable over SSH by the Ansible user, and the user must either be `root` or be able to use `sudo` without a password.

The inventories use `ansible_user=root` and `ansible_become=true`.

### 4.3 OS Compatibility

The repository targets Debian/Ubuntu systems for the package installation tasks and uses the x86_64 architecture for the Kubernetes binaries.

### 4.4 Network Requirements

The environment must satisfy the following connectivity expectations:

- control machine can SSH into all cluster nodes
- etcd nodes can reach each other over peer and client ports
- control-plane nodes can reach etcd at 2379 (HTTPS)
- worker nodes can reach the Kubernetes API server through the VIP or service endpoint
- all nodes must be able to resolve or reach the configured service IPs and API address

---

## 5. Deployment Sequence in Detail

## Stage 1: Deploy HAProxy and Keepalived

This is the first infrastructure layer.

Directory:

- `keepalive_haproxy/`

Command flow:

```bash
cd /root/ansible-playbook/keepalive_haproxy
source ~/ansible-core/bin/activate
ansible-inventory -i inventory.ini --graph
ansible-playbook --syntax-check -i inventory.ini deploy_haproxy_keepalived.yml
ansible-playbook -i inventory.ini deploy_haproxy_keepalived.yml
```

What it does:

- Configures Keepalived VRRP on the three load-balancer nodes
- Creates the VIP 10.168.2.188
- Configures HAProxy to forward traffic to the Kubernetes API servers
- Exposes stats on port 8404

Why it matters:

- The API endpoint must be highly available before the Kubernetes control plane is considered production-ready.
- `kubernetes_api_server_address` in the control-plane inventory points to this VIP.

---

## Stage 2: Deploy etcd

Directory:

- `etcd/`

Command flow:

```bash
cd /root/ansible-playbook/etcd
source ~/ansible-core/bin/activate
ansible-inventory -i inventory.ini --graph
ansible-playbook --syntax-check -i inventory.ini deploy_etcd.yml
ansible-playbook -i inventory.ini deploy_etcd.yml
```

What it does:

- Creates the `etcd` user and group
- Installs required packages
- Downloads the etcd archive
- Installs binaries into `/usr/local/bin`
- Creates the data directory at `/var/lib/etcd`
- Renders the systemd unit from `etcd.service.j2`
- Starts the etcd service

Important note:

- The playbook is designed for an initial cluster creation using `etcd_initial_cluster_state=new`.
- Reusing this on an existing cluster can destroy state unless a migration strategy has been verified.

Verification:

```bash
ETCDCTL_API=3 /usr/local/bin/etcdctl \
  --endpoints=http://10.168.2.108:2379,http://10.168.2.109:2379,http://10.168.2.110:2379 \
  endpoint health
```

This stage is mandatory because the Kubernetes control plane will only become ready when it can talk securely to etcd.

---

## Stage 3: Generate etcd TLS Certificates

Directory:

- `cert/etcd/`

Command flow:

```bash
cd /root/ansible-playbook/cert/etcd
source ~/ansible-core/bin/activate
ansible-playbook --syntax-check -i ../../etcd/inventory.ini issue_etcd_certificates.yml
ansible-playbook -i ../../etcd/inventory.ini issue_etcd_certificates.yml
```

What it creates:

- CA and member certificates under `cert/etcd/generated/`
- Files such as `ca.pem`, `etcd01.pem`, `etcd01-key.pem`, etc.

This stage is crucial because the Kubernetes control-plane playbook uses these certs to configure etcd TLS client access for kube-apiserver.

---

## Stage 4: Generate Kubernetes Control-Plane Certificates

Directory:

- `cert/k8s/master/`

Command flow:

```bash
cd /root/ansible-playbook/cert/k8s/master
source ~/ansible-core/bin/activate
ansible-playbook --syntax-check -i inventory.ini issue_control_plane_certificates.yml
ansible-playbook -i inventory.ini issue_control_plane_certificates.yml
```

What it creates:

- `ca.pem`, `ca-key.pem`
- `apiserver.pem`, `apiserver-key.pem`
- `apiserver-kubelet-client.pem`, `apiserver-kubelet-client-key.pem`
- `front-proxy-ca.pem`, `front-proxy-ca-key.pem`
- `front-proxy-client.pem`, `front-proxy-client-key.pem`
- `admin.pem` and `admin-key.pem` (for admin kubeconfig)
- `controller-manager.pem`, `scheduler.pem`
- `sa.key`, `sa.pub`

This stage is a prerequisite because the control-plane playbook validates the presence of those files and copies them to `/etc/kubernetes/pki`.

---

## Stage 5: Install containerd Runtime

Directory:

- `containers/containerd/`

Command flow:

```bash
cd /root/ansible-playbook/containers/containerd
source ~/ansible-core/bin/activate
ansible-inventory -i inventory.ini --graph
ansible-playbook --syntax-check -i inventory.ini deploy_containerd.yml
ansible-playbook -i inventory.ini deploy_containerd.yml
```

What it does:

- Installs prerequisites and required packages
- Creates directories for containerd
- Downloads the containerd binary archive
- Installs `containerd`, `runc`, and `crictl`
- Writes `/etc/containerd/config.toml`
- Sets systemd cgroup configuration
- Starts and enables the `containerd` systemd service

Why it matters:

- kubelet requires a working container runtime.
- Kubernetes worker nodes will not become Ready without working containerd.

---

## Stage 6: Deploy the Kubernetes Control Plane

Directory:

- `kubernetes/master/`

Command flow:

```bash
cd /root/ansible-playbook/kubernetes/master
source ~/ansible-core/bin/activate
ansible-inventory -i inventory.ini --graph
ansible-playbook --syntax-check -i inventory.ini deploy_control_plane.yml
ansible-playbook -i inventory.ini deploy_control_plane.yml
```

This is the core Kubernetes cluster bootstrap stage.

What the playbook does:

- Validates the cert bundle under `kubernetes_cert_source`
- Validates the ETCD cert bundle under `etcd_cert_source`
- Ensures the etcd TLS client files exist at the expected paths
- Creates system users and directories for Kubernetes
- Downloads the Kubernetes server archive
- Extracts the kube-apiserver, kube-controller-manager, kube-scheduler, and kubectl binaries
- Installs them into `/opt/kubernetes-1.25.16`
- Symlinks them to `/usr/local/bin`
- Copies the PKI files to `/etc/kubernetes/pki`
- Renders `admin.conf`, `controller-manager.conf`, and `scheduler.conf`
- Places the admission configuration file under `/etc/kubernetes/admission-config.yaml`
- Installs systemd unit files for the three control-plane services
- Enables and starts the services
- Validates that the services are active

### 6.1 Control-Plane Component Roles

The control plane includes:

- kube-apiserver: exposes the Kubernetes API, validates requests, and serves API endpoints
- kube-controller-manager: watches cluster state and makes reconciliation decisions
- kube-scheduler: schedules pods to worker nodes

The API server also references the admission configuration stored in `admission-config.yaml`.

### 6.2 Admission Configuration

The repository includes `kubernetes/master/admission-config.yaml`.

This file configures the PodSecurity admission plugin and the default baseline policy, with kube-system exempted, which is necessary for privileged system components such as Calico pods.

This config is copied to:

```text
/etc/kubernetes/admission-config.yaml
```

The API server service uses it via the flag:

```text
--admission-control-config-file=/etc/kubernetes/admission-config.yaml
```

This is an important design decision in a modern Kubernetes v1.25 deployment because the API server needs the correct admission behavior before workloads are scheduled.

### 6.3 etcd TLS Configuration for the API Server

The playbook binds the API server to etcd with:

```ini
kubernetes_etcd_endpoints=https://10.168.2.108:2379,https://10.168.2.109:2379,https://10.168.2.110:2379
kubernetes_etcd_ca_file=/etc/etcd/pki/ca.pem
kubernetes_etcd_cert_file=/etc/etcd/pki/server.pem
kubernetes_etcd_key_file=/etc/etcd/pki/server-key.pem
```

This is essential for TLS-secured etcd access from kube-apiserver.

### 6.4 Kubernetes API Server Address and SANs

The cluster uses:

- API server address: `10.168.2.188`
- Service CIDR: `10.96.0.0/12`
- Cluster DNS: `10.96.0.10`

The certificate generation for apiserver includes SANs such as:

- the VIP address
- the DNS names for the service domain
- Kubernetes in-cluster service addresses
- any extra SANs configured in the inventory

This ensures the certificate matches the API endpoint used by clients.

---

## Stage 7: Deploy Kubernetes Nodes

Directory:

- `kubernetes/node/`

Command flow:

```bash
cd /root/ansible-playbook/kubernetes/node
source ~/ansible-core/bin/activate
ansible-inventory -i inventory.ini --graph
ansible-playbook --syntax-check -i inventory.ini deploy_node_install_uninstall.yml
ansible-playbook -i inventory.ini deploy_node_install_uninstall.yml \
  -e node_install_uninstall_action=install
```

What it does:

- Validates the node certificate bundle on the control machine
- Installs required packages and runtime dependencies
- Downloads kubelet and kube-proxy binaries
- Copies CA and node-specific certificates
- Renders kubeconfigs for kubelet and kube-proxy
- Installs systemd unit files
- Enables and starts kubelet and kube-proxy

This is the place where the worker nodes join the cluster logically by running the Kubernetes node agents.

### 7.1 Important Note About Kubelet v1.25

The repository explicitly warns that in Kubernetes v1.25, the kubelet runtime configuration must remain consistent with the runtime flags. It does not rely on a separate `config.yaml` for runtime settings in the current templates.

This means the runtime endpoint and cgroup settings must be correctly managed in `kubelet.service.j2` and the containerd configuration must be aligned with the data path and socket config.

### 7.2 Worker Node Runtime Requirements

The worker nodes need:

- working containerd socket at `/run/containerd/containerd.sock`
- CA certs from the cluster CA
- node-specific TLS certs for kubelet and kube-proxy
- access to the Kubernetes API server at `10.168.2.188`

Only after these conditions are satisfied can kubelet and kube-proxy become healthy.

---

## 8. Full Deployment Runbook

Below is the recommended complete sequence adapted to the repository as it exists now.

### Step 1: Activate the Ansible environment

```bash
source ~/ansible-core/bin/activate
```

### Step 2: Deploy the HAProxy and Keepalived layer

```bash
cd /root/ansible-playbook/keepalive_haproxy
ansible-playbook -i inventory.ini deploy_haproxy_keepalived.yml
```

### Step 3: Deploy etcd

```bash
cd /root/ansible-playbook/etcd
ansible-playbook -i inventory.ini deploy_etcd.yml
```

### Step 4: Generate etcd TLS certs

```bash
cd /root/ansible-playbook/cert/etcd
ansible-playbook -i ../../etcd/inventory.ini issue_etcd_certificates.yml
```

### Step 5: Generate Kubernetes control-plane TLS certs

```bash
cd /root/ansible-playbook/cert/k8s/master
ansible-playbook -i inventory.ini issue_control_plane_certificates.yml
```

### Step 6: Install containerd

```bash
cd /root/ansible-playbook/containers/containerd
ansible-playbook -i inventory.ini deploy_containerd.yml
```

### Step 7: Deploy Kubernetes control plane

```bash
cd /root/ansible-playbook/kubernetes/master
ansible-playbook -i inventory.ini deploy_control_plane.yml
```

### Step 8: Deploy Kubernetes nodes

```bash
cd /root/ansible-playbook/kubernetes/node
ansible-playbook -i inventory.ini deploy_node_install_uninstall.yml \
  -e node_install_uninstall_action=install
```

---

## 9. Validation Checks

After all stages are completed, the following checks should be run.

### 9.1 Check the etcd cluster health

```bash
ETCDCTL_API=3 /usr/local/bin/etcdctl \
  --endpoints=http://10.168.2.108:2379,http://10.168.2.109:2379,http://10.168.2.110:2379 \
  endpoint health
```

### 9.2 Check the Kubernetes control-plane services

On a control-plane node:

```bash
systemctl status kube-apiserver kube-controller-manager kube-scheduler
```

### 9.3 Check the API server readiness

```bash
kubectl --kubeconfig=/etc/kubernetes/admin.conf version --client
kubectl --kubeconfig=/etc/kubernetes/admin.conf get --raw='/readyz?verbose'
```

### 9.4 Check listening ports

```bash
ss -lntp | grep -E ':(6443|10257|10259)'
```

### 9.5 Check kubelet and kube-proxy status on nodes

```bash
systemctl status kubelet kube-proxy
```

### 9.6 Check cluster node registration

```bash
kubectl --kubeconfig=/etc/kubernetes/admin.conf get nodes -o wide
```

### 9.7 Check pod readiness

```bash
kubectl --kubeconfig=/etc/kubernetes/admin.conf get pods -A
```

---

## 10. Troubleshooting Guidance

### 10.1 etcd Health Problems

Symptoms:

- kube-apiserver fails to start
- control-plane services remain unhealthy
- etcd endpoint health shows errors

Actions:

```bash
sudo systemctl status etcd
sudo journalctl -u etcd -n 200 --no-pager
```

If the cluster was rebuilt incorrectly or the data directory remains stale:

```bash
sudo systemctl stop etcd
sudo rm -rf /var/lib/etcd/*
sudo systemctl start etcd
```

### 10.2 Kubernetes APIServer Fails to Start

Symptoms:

- service exits immediately
- TLS validation fails
- etcd connectivity errors

Checks:

```bash
journalctl -u kube-apiserver -n 200 --no-pager
ls -l /etc/kubernetes/pki
ls -l /etc/etcd/pki
```

Important points:

- the API server certificate SANs must include the VIP address and internal service names
- the kube-apiserver must trust the etcd certificates and the etcd service endpoint must be reachable over HTTPS
- the admission config file and kubeconfig files must exist and be readable

### 10.3 Worker Node Does Not Join

Symptoms:

- node is NotReady
- kubelet or kube-proxy fails
- containerd runtime is not reachable

Checks:

```bash
journalctl -u kubelet -n 200 --no-pager
journalctl -u kube-proxy -n 200 --no-pager
ss -lntp | grep 10250
ls -l /etc/kubernetes/pki
```

Also verify the kubeconfig and the API endpoint are correct.

### 10.4 Containerd Problems

Symptoms:

- kubelet cannot start container runtime
- CRI socket is missing
- containerd service fails

Checks:

```bash
systemctl status containerd
journalctl -u containerd -n 200 --no-pager
crictl --runtime-endpoint unix:///run/containerd/containerd.sock ps
```

---

## 11. Important Operational Notes

### 11.1 The Repository Does Not Include a CNI Deployment

This repository installs the control-plane binaries, node binaries, and containerd runtime, but it does not include a full container networking deployment playbook in the same way it includes the runtime and control-plane installation tasks.

That means a CNI solution, such as Calico, still needs to be deployed separately after the cluster is ready and the control plane is running.

### 11.2 Kubernetes Version Alignment

The current configuration is explicitly aligned to Kubernetes v1.25.16 and related runtime versions.

The relevant version pins include:

- Kubernetes version: 1.25.16
- containerd version: 1.6.24
- runc version: 1.1.10
- crictl version: 1.25.0
- etcd version: 3.5.16

Version consistency matters. These components must align with the expected Kubernetes API compatibility and runtime behavior.

### 11.3 Do Not Re-run Cluster Creation Anew Without a Verified Plan

The etcd playbook is intentionally designed for initial creation. If the data directory is removed or the cluster is reinitialized in the same environment without a validated migration or rebuild plan, the cluster state may be lost.

---

## 12. Summary of the Deployment Logic

The repository implements an HA Kubernetes deployment by layering responsibilities in this order:

1. Provide the API-fronting load balancer.
2. Build the etcd data cluster.
3. Generate the TLS material required by etcd and Kubernetes.
4. Install the runtime for all nodes.
5. Start the Kubernetes control plane.
6. Start kubelet and kube-proxy on worker nodes.
7. Validate readiness and fix certificate or service-level issues.

This is the correct logic for a modern Kubernetes v1.25 deployment built with Ansible and binaries instead of a full managed service abstraction.

---

## 13. Practical Deployment Checklist

Use this checklist before declaring the cluster ready:

- [ ] HAProxy and Keepalived are installed and the VIP is active
- [ ] etcd cluster is healthy and the member list is valid
- [ ] etcd TLS certificates exist and match the target hosts
- [ ] Kubernetes control-plane certificates exist in the generated bundle
- [ ] containerd is installed and running on all required hosts
- [ ] kube-apiserver, kube-controller-manager, and kube-scheduler are active
- [ ] API server returns healthy status
- [ ] kubelet and kube-proxy are running on worker nodes
- [ ] worker nodes are registered in the cluster
- [ ] a CNI plugin is installed and pod networking is active

---

## 14. Conclusion

This repository is structured to deploy a Kubernetes v1.25 cluster in a disciplined, layered, and production-oriented fashion. The critical dependency chain is:

- network + VIP
- etcd
- certificates
- runtime
- control plane
- node agents

Following the exact order in this document is the safest way to build a healthy cluster from the current repository state.
