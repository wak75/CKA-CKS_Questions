# RBAC — Role-Based Access Control in Kubernetes

## What is RBAC?

RBAC (Role-Based Access Control) is a method of regulating access to Kubernetes resources based on the roles assigned to users, groups, or service accounts. It answers the question:

> **"Who can do what on which resources?"**

RBAC is enabled by default in Kubernetes 1.8+ and is the recommended authorization mechanism.

---

## Core Concepts

### Subject
Who is requesting access:
- **User** — a human user (managed outside Kubernetes, e.g., via certificates or OIDC)
- **Group** — a collection of users
- **ServiceAccount** — an identity for processes running inside Pods

### Resource
What Kubernetes object is being accessed:
- Pods, Deployments, Services, ConfigMaps, Secrets, Nodes, etc.

### Verb
What action is being performed:
| Verb | HTTP Method | Description |
|---|---|---|
| `get` | GET | Read a single resource |
| `list` | GET (collection) | List multiple resources |
| `watch` | GET (watch) | Stream changes to resources |
| `create` | POST | Create a new resource |
| `update` | PUT | Replace a resource |
| `patch` | PATCH | Partially update a resource |
| `delete` | DELETE | Delete a resource |
| `deletecollection` | DELETE (collection) | Delete multiple resources |

---

## RBAC API Objects

### 1. Role
Grants permissions **within a specific namespace**.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: dev
rules:
  - apiGroups: [""]          # "" = core API group
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

### 2. ClusterRole
Grants permissions **cluster-wide** (across all namespaces) or for non-namespaced resources (Nodes, PersistentVolumes, Namespaces).

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
```

### 3. RoleBinding
Binds a Role (or ClusterRole) to a subject **within a namespace**.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: dev
subjects:
  - kind: User
    name: alice
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### 4. ClusterRoleBinding
Binds a ClusterRole to a subject **across all namespaces**.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-reader-binding
subjects:
  - kind: User
    name: bob
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## Binding Combinations

| Role Type | Binding Type | Scope |
|---|---|---|
| Role | RoleBinding | Single namespace only |
| ClusterRole | RoleBinding | Single namespace (reuse cluster role in namespace) |
| ClusterRole | ClusterRoleBinding | All namespaces + cluster-scoped resources |

> A ClusterRole bound with a RoleBinding gives namespace-scoped permissions only — it does NOT grant cluster-wide access.

---

## Common Use Cases

### 1. Grant a developer read-only access to Pods in a namespace

```yaml
# Role
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: dev-pod-reader
  namespace: staging
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-pod-reader-binding
  namespace: staging
subjects:
  - kind: User
    name: developer1
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: dev-pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### 2. Grant a ServiceAccount permission to manage Deployments

```yaml
# ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: deploy-manager
  namespace: production
---
# Role
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployment-manager
  namespace: production
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "create", "update", "patch", "delete"]
---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: deployment-manager-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: deploy-manager
    namespace: production
roleRef:
  kind: Role
  name: deployment-manager
  apiGroup: rbac.authorization.k8s.io
```

### 3. Grant a group cluster-admin access

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-group-binding
subjects:
  - kind: Group
    name: platform-admins
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

---

## Built-in ClusterRoles

Kubernetes ships with several default ClusterRoles:

| ClusterRole | Description |
|---|---|
| `cluster-admin` | Full access to everything |
| `admin` | Full access within a namespace |
| `edit` | Read/write most resources in a namespace |
| `view` | Read-only access in a namespace |

```bash
# View all built-in cluster roles
kubectl get clusterroles
```

---

## Attaching RBAC to a ServiceAccount in a Pod

Pods use a ServiceAccount to authenticate to the API server. To give a Pod RBAC permissions:

```yaml
# 1. Create ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: default
---
# 2. Create Role
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: configmap-reader
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list"]
---
# 3. Bind Role to ServiceAccount
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: configmap-reader-binding
  namespace: default
subjects:
  - kind: ServiceAccount
    name: app-sa
    namespace: default
roleRef:
  kind: Role
  name: configmap-reader
  apiGroup: rbac.authorization.k8s.io
---
# 4. Attach ServiceAccount to Pod
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  namespace: default
spec:
  serviceAccountName: app-sa
  containers:
    - name: app
      image: myapp:1.0
```

---

## Useful kubectl Commands

```bash
# Create a role imperatively
kubectl create role pod-reader \
  --verb=get,list,watch \
  --resource=pods \
  --namespace=dev

# Create a clusterrole
kubectl create clusterrole node-reader \
  --verb=get,list,watch \
  --resource=nodes

# Create a rolebinding
kubectl create rolebinding read-pods-binding \
  --role=pod-reader \
  --user=alice \
  --namespace=dev

# Create a clusterrolebinding
kubectl create clusterrolebinding node-reader-binding \
  --clusterrole=node-reader \
  --user=bob

# Check if a user can perform an action
kubectl auth can-i get pods --namespace=dev --as=alice

# Check as a ServiceAccount
kubectl auth can-i list secrets \
  --namespace=production \
  --as=system:serviceaccount:production:deploy-manager

# List all roles in a namespace
kubectl get roles -n dev
kubectl get rolebindings -n dev

# List cluster-scoped
kubectl get clusterroles
kubectl get clusterrolebindings

# Describe a role to see its rules
kubectl describe role pod-reader -n dev
kubectl describe clusterrole cluster-admin
```

---

## apiGroups Reference

| Resource | apiGroup |
|---|---|
| Pods, Services, ConfigMaps, Secrets, Nodes | `""` (core) |
| Deployments, ReplicaSets, StatefulSets, DaemonSets | `"apps"` |
| Jobs, CronJobs | `"batch"` |
| Ingresses, NetworkPolicies | `"networking.k8s.io"` |
| HorizontalPodAutoscalers | `"autoscaling"` |
| PodDisruptionBudgets | `"policy"` |
| ClusterRoles, Roles, RoleBindings | `"rbac.authorization.k8s.io"` |
| CustomResourceDefinitions | `"apiextensions.k8s.io"` |

---

## Benefits of RBAC

- **Least privilege** — grant only the permissions needed, nothing more
- **Namespace isolation** — teams can be scoped to their own namespace
- **Auditability** — clear declaration of who can do what
- **Dynamic** — changes take effect immediately without restarting components
- **ServiceAccount integration** — workloads can have their own identities
- **Reusability** — ClusterRoles can be reused across namespaces via RoleBindings

---

## Limitations

| Limitation | Detail |
|---|---|
| **No deny rules** | RBAC is additive only — you grant permissions, you cannot explicitly deny them |
| **No resource-level filtering** | You cannot restrict access to a specific Pod by name (e.g., only Pod `nginx-1`). You can restrict by `resourceNames` but it is limited |
| **No field-level access control** | Cannot restrict access to specific fields within a resource |
| **User management is external** | Kubernetes does not manage users — certificates, OIDC, or external providers handle that |
| **No time-based access** | Cannot grant temporary or time-limited permissions natively |
| **Complexity at scale** | Many roles and bindings become hard to audit without tooling |

### resourceNames — restricting by name (limited)

```yaml
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["app-config"]   # only this specific ConfigMap
    verbs: ["get"]
```

---

## RBAC Security Best Practices

- Always follow the **principle of least privilege**
- Avoid using `cluster-admin` for applications or developers
- Prefer `Role` over `ClusterRole` unless cluster-wide access is genuinely needed
- Avoid `*` (wildcard) verbs or resources in production
- Audit bindings regularly with `kubectl get rolebindings,clusterrolebindings -A`
- Use dedicated ServiceAccounts per application — do not use the `default` ServiceAccount
- Use `kubectl auth can-i` to verify permissions before and after changes

---

## Interview Questions

### Conceptual

**Q: What is RBAC in Kubernetes?**
A: RBAC is a method of controlling access to Kubernetes API resources based on roles assigned to users, groups, or service accounts. It defines who (subject) can perform what actions (verbs) on which resources.

**Q: What is the difference between a Role and a ClusterRole?**
A: A Role is namespaced — it grants permissions within a specific namespace. A ClusterRole is cluster-scoped — it grants permissions across all namespaces or for non-namespaced resources like Nodes and PersistentVolumes.

**Q: What is the difference between a RoleBinding and a ClusterRoleBinding?**
A: A RoleBinding grants permissions within a single namespace. A ClusterRoleBinding grants permissions cluster-wide. Importantly, a ClusterRole can be bound with a RoleBinding to restrict it to a single namespace.

**Q: Can RBAC deny access explicitly?**
A: No. RBAC is purely additive — you can only grant permissions, not deny them. If no rule grants an action, it is implicitly denied.

**Q: What are the four RBAC API objects?**
A: Role, ClusterRole, RoleBinding, and ClusterRoleBinding.

**Q: What is a ServiceAccount and how does it relate to RBAC?**
A: A ServiceAccount is an identity for processes running inside Pods. RBAC roles are bound to ServiceAccounts to give Pods specific permissions to interact with the Kubernetes API.

**Q: What does `apiGroups: [""]` mean in a Role?**
A: It refers to the core API group, which contains resources like Pods, Services, ConfigMaps, Secrets, and Nodes.

**Q: What is the `kubectl auth can-i` command used for?**
A: It checks whether a user or service account has permission to perform a specific action on a resource. Useful for verifying RBAC configuration.

```bash
kubectl auth can-i delete pods --namespace=prod --as=alice
```

**Q: What happens if a Pod does not have a serviceAccountName specified?**
A: It uses the `default` ServiceAccount of its namespace, which typically has minimal permissions.

**Q: Can a ClusterRole bound with a RoleBinding give cluster-wide access?**
A: No. When a ClusterRole is bound using a RoleBinding (not ClusterRoleBinding), it only grants access within the namespace of the RoleBinding.

### Scenario-Based

**Q: A CI/CD pipeline needs to deploy to the `production` namespace but should not access any other namespace. How would you set this up?**
A: Create a ServiceAccount in `production`, create a Role in `production` with verbs for deployments, and bind the Role to the ServiceAccount with a RoleBinding scoped to `production`.

**Q: A developer reports they cannot list Pods in the `staging` namespace. How do you debug this?**
A:
```bash
# Check their permissions
kubectl auth can-i list pods --namespace=staging --as=developer1

# Check what roles are bound to them
kubectl get rolebindings -n staging -o yaml | grep developer1

# Describe the binding
kubectl describe rolebinding <binding-name> -n staging
```

**Q: You want to give a monitoring tool (Prometheus) read access to Pods and Services across all namespaces. What is the correct approach?**
A: Create a ClusterRole with `get`, `list`, `watch` on `pods` and `services`. Create a ServiceAccount for Prometheus. Bind them with a ClusterRoleBinding. Attach the ServiceAccount to the Prometheus Pod.

**Q: How would you prevent a user from accidentally deleting Deployments in production?**
A: Ensure the user's Role or ClusterRole in the `production` namespace does not include the `delete` verb for `deployments`. Since RBAC has no deny rules, simply omit `delete` from any role bound to them in that namespace.

**Q: What is the risk of using `verbs: ["*"]` in a Role?**
A: It grants all possible actions (get, list, watch, create, update, patch, delete, deletecollection) on the specified resources. This violates the principle of least privilege and can lead to accidental or malicious data loss.

---

## Summary

| Concept | Key Point |
|---|---|
| Role | Namespace-scoped permissions |
| ClusterRole | Cluster-wide permissions |
| RoleBinding | Attaches Role/ClusterRole to subject in a namespace |
| ClusterRoleBinding | Attaches ClusterRole to subject cluster-wide |
| Subject types | User, Group, ServiceAccount |
| No deny rules | RBAC is additive only |
| Least privilege | Always grant minimum required permissions |
| Debug tool | `kubectl auth can-i` |
