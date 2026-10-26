# Service Accounts in Kubernetes

## What is a Service Account?

A Service Account is a **non-human identity** used by processes running inside Pods to authenticate with the Kubernetes API server. While regular user accounts are for humans, service accounts are for applications and workloads.

> **Think of it as:** A Pod's identity card when talking to the Kubernetes API.

Every Pod in Kubernetes runs with a service account. If you don't specify one, it uses the `default` service account of its namespace.

---

## Why Service Accounts Exist

Pods often need to interact with the Kubernetes API — to list other Pods, read ConfigMaps, create Jobs, etc. Service accounts provide:

- A verifiable identity for the Pod
- A token the Pod uses to authenticate API requests
- A way to attach RBAC permissions to a workload

---

## How Service Accounts Work

1. A ServiceAccount is created in a namespace
2. Kubernetes automatically creates a token (JWT) for it
3. When a Pod uses that ServiceAccount, the token is mounted into the Pod at:
   `/var/run/secrets/kubernetes.io/serviceaccount/token`
4. The Pod uses this token in API requests (`Authorization: Bearer <token>`)
5. The API server verifies the token and checks RBAC rules

Also mounted automatically:
- `/var/run/secrets/kubernetes.io/serviceaccount/ca.crt` — cluster CA certificate
- `/var/run/secrets/kubernetes.io/serviceaccount/namespace` — current namespace name

---

## Default Service Account

Every namespace gets a `default` service account automatically. Pods that don't specify a `serviceAccountName` use it.

```bash
kubectl get serviceaccounts -n default
# NAME      SECRETS   AGE
# default   0         10d
```

The `default` service account has **no RBAC permissions** by default (beyond basic API discovery). It is a security best practice to create dedicated service accounts per application.

---

## Creating a Service Account

### Imperative

```bash
kubectl create serviceaccount my-app-sa -n production
```

### Declarative

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: production
```

```bash
kubectl apply -f serviceaccount.yaml
```

---

## Attaching a Service Account to a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  namespace: production
spec:
  serviceAccountName: my-app-sa   # attach the service account here
  containers:
    - name: app
      image: myapp:1.0
```

For a Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      serviceAccountName: my-app-sa   # set in Pod template spec
      containers:
        - name: app
          image: myapp:1.0
```

---

## Granting Permissions to a Service Account (RBAC)

A service account alone has no permissions. You must bind it to a Role or ClusterRole.

```yaml
# 1. ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: config-reader-sa
  namespace: staging
---
# 2. Role
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: configmap-reader
  namespace: staging
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]
---
# 3. RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: config-reader-binding
  namespace: staging
subjects:
  - kind: ServiceAccount
    name: config-reader-sa
    namespace: staging
roleRef:
  kind: Role
  name: configmap-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## Consuming the Service Account Token Inside a Pod

The token is automatically mounted. Any process inside the Pod can use it:

```bash
# Inside the container
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
NAMESPACE=$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)
CACERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

# Call the Kubernetes API
curl -s --cacert $CACERT \
  -H "Authorization: Bearer $TOKEN" \
  https://kubernetes.default.svc/api/v1/namespaces/$NAMESPACE/pods
```

---

## Disabling Auto-Mounting of the Token

If a Pod does not need to call the Kubernetes API, disable token auto-mounting to reduce the attack surface:

### On the Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: no-api-access
spec:
  automountServiceAccountToken: false
  containers:
    - name: app
      image: myapp:1.0
```

### On the ServiceAccount (applies to all Pods using it)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: no-api-sa
  namespace: default
automountServiceAccountToken: false
```

> Pod-level setting overrides ServiceAccount-level setting.

---

## Token Projection (Kubernetes 1.12+)

Modern Kubernetes uses **projected service account tokens** — they are:
- **Audience-bound** — only valid for a specific audience
- **Time-bound** — expire after a configurable duration (default 1 hour)
- **Pod-bound** — invalid if the Pod is deleted

```yaml
spec:
  volumes:
    - name: token-vol
      projected:
        sources:
          - serviceAccountToken:
              path: token
              expirationSeconds: 3600
              audience: my-api
  containers:
    - name: app
      image: myapp:1.0
      volumeMounts:
        - name: token-vol
          mountPath: /var/run/secrets/tokens
```

---

## ImagePullSecrets with Service Accounts

A service account can carry image pull secrets so all Pods using it automatically get registry credentials:

```bash
# Create registry secret
kubectl create secret docker-registry my-registry-secret \
  --docker-server=registry.example.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  -n production
```

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: production
imagePullSecrets:
  - name: my-registry-secret
```

---

## Real-World Use Cases

| Use Case | Description |
|---|---|
| **CI/CD pipelines** | Jenkins/ArgoCD Pod needs to create/update Deployments |
| **Monitoring tools** | Prometheus needs to list Pods and Services across namespaces |
| **Operators & controllers** | Custom controllers watch and manage resources |
| **Vault / secret injection** | Pod authenticates to HashiCorp Vault using its SA token |
| **Cloud provider IAM** | AWS IRSA / GCP Workload Identity links SA to cloud IAM roles |
| **Internal microservices** | Service A authenticates to Kubernetes API to discover Service B |

---

## Service Account vs User Account

| | Service Account | User Account |
|---|---|---|
| **For** | Processes/Pods | Humans |
| **Managed by** | Kubernetes | External (certs, OIDC, LDAP) |
| **Namespace-scoped** | Yes | No (cluster-scoped) |
| **Created with** | `kubectl create sa` | External tooling |
| **Token** | Auto-mounted in Pod | Obtained via kubeconfig |
| **RBAC binding subject kind** | `ServiceAccount` | `User` or `Group` |

---

## Useful kubectl Commands

```bash
# Create service account
kubectl create serviceaccount my-sa -n dev

# List service accounts
kubectl get serviceaccounts -n dev

# Describe a service account
kubectl describe serviceaccount my-sa -n dev

# Get the token for a service account (Kubernetes 1.24+)
kubectl create token my-sa -n dev

# Get the token with custom expiry
kubectl create token my-sa -n dev --duration=24h

# Check what a service account can do
kubectl auth can-i list pods \
  --as=system:serviceaccount:dev:my-sa \
  -n dev

# View token mounted inside a running pod
kubectl exec -it <pod-name> -- \
  cat /var/run/secrets/kubernetes.io/serviceaccount/token

# Get all pods using a specific service account
kubectl get pods -n dev -o jsonpath=\
  '{range .items[?(@.spec.serviceAccountName=="my-sa")]}{.metadata.name}{"\n"}{end}'
```

---

## Kubernetes 1.24 Token Changes

Before 1.24: Kubernetes auto-created a long-lived Secret with a token for every ServiceAccount.

After 1.24: Tokens are no longer stored in Secrets by default. Instead:
- Tokens are projected directly into Pods (time-limited, auto-rotated)
- Use `kubectl create token <sa-name>` to get a token manually
- To create a long-lived token (not recommended), create a Secret manually:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-sa-token
  namespace: default
  annotations:
    kubernetes.io/service-account.name: my-sa
type: kubernetes.io/service-account-token
```

---

## Security Best Practices

- Create **dedicated service accounts** per application — never share them
- Disable `automountServiceAccountToken: false` for Pods that don't need API access
- Follow **least privilege** — only grant the minimum verbs and resources needed
- Never use the `default` service account for application workloads
- Rotate tokens regularly using projected token volumes with short expiry
- Audit service account permissions with `kubectl auth can-i --as=system:serviceaccount:...`
- Avoid binding service accounts to `cluster-admin` unless absolutely necessary

---

## Interview Questions

### Conceptual

**Q: What is a service account in Kubernetes?**
A: A service account is a namespaced Kubernetes object that provides an identity for processes running inside Pods. It allows Pods to authenticate with the Kubernetes API server and is used to attach RBAC permissions to workloads.

**Q: What is the difference between a user account and a service account?**
A: User accounts are for humans and are managed outside Kubernetes (via certificates or OIDC). Service accounts are for Pods/processes, are namespace-scoped, are managed by Kubernetes, and have their tokens auto-mounted into Pods.

**Q: What happens if you don't specify a serviceAccountName in a Pod spec?**
A: The Pod uses the `default` service account of its namespace. The default service account has no RBAC permissions beyond basic API discovery.

**Q: Where is the service account token mounted inside a Pod?**
A: At `/var/run/secrets/kubernetes.io/serviceaccount/token`. The CA certificate is at `.../ca.crt` and the namespace at `.../namespace`.

**Q: How does a Pod use its service account to call the Kubernetes API?**
A: It reads the token from the mounted path and includes it in the `Authorization: Bearer <token>` header when making HTTP requests to `https://kubernetes.default.svc`.

**Q: What changed with service account tokens in Kubernetes 1.24?**
A: Before 1.24, Kubernetes auto-created a long-lived Secret token for every ServiceAccount. From 1.24 onward, tokens are projected directly into Pods — they are time-limited (1 hour default), audience-bound, and auto-rotated. No Secret is created automatically.

**Q: How do you disable automatic token mounting for a Pod?**
A: Set `automountServiceAccountToken: false` in the Pod spec or on the ServiceAccount itself. Pod-level setting takes precedence.

**Q: Can a service account in namespace A access resources in namespace B?**
A: Not by default. You would need to create a ClusterRoleBinding that binds the service account to a ClusterRole with permissions in namespace B, or create a RoleBinding in namespace B that references the service account from namespace A.

**Q: What is the format of a service account identity in RBAC subjects?**
A: `system:serviceaccount:<namespace>:<service-account-name>`
Example: `system:serviceaccount:production:my-app-sa`

**Q: How do you get a token for a service account to use outside the cluster?**
A: Use `kubectl create token <sa-name> -n <namespace>`. This generates a short-lived token. For long-lived tokens (not recommended), create a Secret of type `kubernetes.io/service-account-token`.

### Scenario-Based

**Q: A Pod running in the `monitoring` namespace needs to list Pods across all namespaces. How do you set this up?**
A:
```bash
# 1. Create service account
kubectl create sa prometheus-sa -n monitoring

# 2. Create ClusterRole
kubectl create clusterrole pod-lister \
  --verb=get,list,watch --resource=pods

# 3. Bind with ClusterRoleBinding
kubectl create clusterrolebinding prometheus-pod-lister \
  --clusterrole=pod-lister \
  --serviceaccount=monitoring:prometheus-sa

# 4. Use in Deployment with serviceAccountName: prometheus-sa
```

**Q: You notice a Pod can delete Secrets in its namespace even though it shouldn't. How do you investigate?**
A:
```bash
# Find the service account used by the Pod
kubectl get pod <pod-name> -o jsonpath='{.spec.serviceAccountName}'

# Check what it can do
kubectl auth can-i delete secrets \
  --as=system:serviceaccount:<namespace>:<sa-name> \
  -n <namespace>

# Find bindings for this service account
kubectl get rolebindings,clusterrolebindings -A -o yaml \
  | grep -A5 <sa-name>
```
Then remove the `delete` verb from the Role granting it.

**Q: An application gets a 403 Forbidden when trying to read a ConfigMap. What do you check?**
A:
1. Confirm which service account the Pod uses: `kubectl get pod <name> -o jsonpath='{.spec.serviceAccountName}'`
2. Check if a Role/RoleBinding exists granting `get` on `configmaps`
3. Run `kubectl auth can-i get configmaps --as=system:serviceaccount:<ns>:<sa>`
4. If missing, create the Role and RoleBinding

**Q: You want all Pods in a namespace to pull images from a private registry without specifying imagePullSecrets on every Pod. How?**
A: Attach the `imagePullSecrets` to the namespace's service account (or a custom one used by all Pods). Kubernetes automatically adds those pull secrets to any Pod using that service account.

**Q: A developer asks why their Pod has a token at `/var/run/secrets/kubernetes.io/serviceaccount/token` even though they never created a service account. Why?**
A: Every namespace has a `default` service account. Any Pod that doesn't specify `serviceAccountName` uses it, and by default `automountServiceAccountToken` is `true`, so the token is always mounted unless explicitly disabled.

---

## Summary

| Concept | Key Point |
|---|---|
| What it is | Non-human identity for Pods |
| Default SA | Every namespace has one; used if none specified |
| Token location | `/var/run/secrets/kubernetes.io/serviceaccount/token` |
| RBAC subject format | `system:serviceaccount:<namespace>:<name>` |
| Permissions | None by default — attach via RoleBinding/ClusterRoleBinding |
| Disable mounting | `automountServiceAccountToken: false` |
| 1.24+ tokens | Time-limited, projected, no auto-created Secret |
| Best practice | One SA per application, least privilege, disable if not needed |
