# End-User Configuration for Kubernetes Access

This document describes how to prepare Kubernetes access for end users and how to avoid handing out full cluster certificates. The direct `kubectl` setup below is useful for a controlled lab or troubleshooting workflow, but it is not the recommended model for production user access.

## 1. Direct kubeconfig setup (simple but not recommended for end users)

If you are building a temporary admin kubeconfig for a user or operator, this is the classic pattern:

```bash
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ca.pem \
  --server=https://10.168.2.188:6443 \
  --embed-certs=true

kubectl config set-credentials admin \
  --client-certificate=/etc/kubernetes/admin.pem \
  --client-key=/etc/kubernetes/admin-key.pem \
  --embed-certs=true

kubectl config set-context kubernetes \
  --cluster=kubernetes \
  --user=admin

kubectl config use-context kubernetes
```

This creates a `kubectl` context using the cluster CA and a client certificate for the `admin` identity. It is straightforward and works well when:

- you trust the user with cluster-level access,
- you are in a lab environment,
- the user is a cluster administrator or an operator.

However, this approach usually means sharing the cluster CA and user certificate material, which is not ideal for normal user access.

## 2. Important security consideration

Before exposing access to end users, do not hand out all cluster credentials by default. In practice, the cluster CA, admin certs, and private keys are highly sensitive. The cluster CA is the trust anchor for the control plane and should not be distributed widely.

A safer model is:

- do not provide all cluster certificates to every user;
- do not use the cluster's root CA as the general user identity trust source;
- grant only the minimum RBAC permissions required;
- use a separate identity method for human users.

This is the reason many clusters use a dedicated identity provider instead of sharing cluster admin certificates.

## 3. Recommended alternative: external identity + RBAC

The usual and better design is to use an external identity system such as:

- OIDC (for example, Keycloak, Dex, Azure AD, Google, GitHub OIDC, etc.);
- LDAP or Active Directory;
- a separate internal certificate authority for user identities.

This approach avoids distributing the entire Kubernetes control-plane certificate set to end users. Instead, users authenticate through a trusted identity provider, and Kubernetes authorizes them with RBAC rules.

### Typical pattern

1. Configure the Kubernetes API server to trust the external identity provider.
2. Create RBAC roles and bindings for users or groups.
3. Provide each user with a kubeconfig that points to the cluster but does not include cluster private keys.
4. Use `--certificate-authority` only for the public CA certificate if needed.
5. Let user authentication occur using their identity provider token or certificate.

This is often described as:

- external identity integration,
- OIDC-based authentication,
- delegated authentication,
- user certificate authentication with a separate CA,
- or a dedicated user CA rather than the cluster CA.

## 4. What if we still want a certificate-based approach?

If certificates are required, the better pattern is to issue user certificates from a separate CA, not from the cluster's internal control-plane trust chain.

Example concept:

- cluster CA: used for Kubernetes control-plane components and internal trust;
- user CA: used for end-user identity certificates;
- RBAC: decides what that user can do in the cluster.

In this model, the user receives only their own client certificate and key, not the full cluster certificate bundle. The certificate is associated with the user identity and is constrained by RBAC permissions.

This keeps the certificate trust boundary clear:

- cluster certificates are for cluster components and internal services;
- user certificates are for end-user access;
- Kubernetes RBAC enforces authorization.

## 5. Example of a safer kubeconfig for a user

If you are using a user certificate from a dedicated CA, the kubeconfig can be limited to the user identity and a public cluster CA certificate, not the full internal cluster material.

```bash
kubectl config set-cluster kubernetes \
  --server=https://10.168.2.188:6443 \
  --certificate-authority=/path/to/public-cluster-ca.pem \
  --embed-certs=true

kubectl config set-credentials alice \
  --client-certificate=/path/to/alice.crt \
  --client-key=/path/to/alice.key \
  --embed-certs=true

kubectl config set-context alice-context \
  --cluster=kubernetes \
  --user=alice \
  --namespace=team-a

kubectl config use-context alice-context
```

This keeps user authentication separate from infrastructure trust and is much safer than handing out the cluster's admin bundle.

## 6. Recommended RBAC example

After user identity is established, the cluster should grant only the permissions needed. For example:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-reader
  namespace: team-a
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alice-read-access
  namespace: team-a
subjects:
- kind: User
  name: alice
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: app-reader
  apiGroup: rbac.authorization.k8s.io
```

This enforces least privilege and ensures that user access is controlled by Kubernetes authorization rather than by shared admin credentials.

## 7. Best practice summary

For end-user access, the preferred approach is:

- use a separate identity provider or dedicated user CA;
- avoid sharing the cluster's admin and CA materials;
- grant only scoped RBAC permissions;
- keep cluster certificates for cluster components and in-cluster trust; 
- provide end users with their own identity and minimal access rights.

If you need a short operational recommendation:

> Do not hand out the cluster CA and admin certs to normal users. Use OIDC, LDAP, or a dedicated user CA, then apply RBAC to authorize only the required permissions.

## 8. Final guidance

For a small lab or troubleshooting environment, the direct admin kubeconfig pattern is acceptable. For real user management, the recommended design is identity federation or a dedicated user certificate CA, combined with strict RBAC. This is the principle behind secure end-user access in Kubernetes.
