# Nodes in Kubernetes

## What is a Node?

A Node is a worker machine in Kubernetes — physical or virtual — where Pods are scheduled and run. Every node is managed by the control plane and runs three key components:

| Component | Role |
|---|---|
| **kubelet** | Agent that communicates with the API server, manages Pods on the node |
| **kube-proxy** | Manages network rules for Service routing |
| **Container runtime** | Runs containers (containerd, CRI-O, Docker) |

---

## Types of Nodes

| Type | Role |
|---|---|
| **Control Plane node** | Runs API server, scheduler, controller manager, etcd |
| **Worker node** | Runs application Pods |

---

## How to Add a Node to a Cluster

### Using kubeadm

#### Step 1 — On the control plane, generate a join token

```bash
# Generate a new join command (token valid for 24h)
kubeadm token create --print-join-command
```

Output:
```
kubeadm join 192.168.1.100:6443 \
  --token abc123.xyz456 \
  --discovery-token-ca-cert-hash sha256:<hash>
```

#### Step 2 — On the new worker node, run the join command

```bash
# Install prerequisites first (kubelet, kubeadm, kubectl, container runtime)
# Then run the join command from step 1
kubeadm join 192.168.1.100:6443 \
  --token abc123.xyz456 \
  --discovery-token-ca-cert-hash sha256:<hash>
```

#### Step 3 — Verify on the control plane

```bash
kubectl get nodes
# NAME           STATUS   ROLES           AGE   VERSION
# controlplane   Ready    control-plane   10d   v1.28.0
# worker-1       Ready    <none>          1m    v1.28.0
```

### Listing existing tokens

```bash
kubeadm token list
```

### If the token has expired, create a new one

```bash
kubeadm token create --print-join-command
```

---

## Node Status and Information

```bash
# List all nodes
kubectl get nodes

# List with more details (IP, OS, kernel)
kubectl get nodes -o wide

# Describe a node (conditions, capacity, allocated resources, events)
kubectl describe node <node-name>

# Check node labels
kubectl get nodes --show-labels

# Get node resource usage
kubectl top nodes
```

### Node Conditions

| Condition | Meaning |
|---|---|
| `Ready` | Node is healthy and ready to accept Pods |
| `MemoryPressure` | Node is low on memory |
| `DiskPressure` | Node is low on disk space |
| `PIDPressure` | Too many processes on the node |
| `NetworkUnavailable` | Network not configured correctly |

---

## How to Schedule a Resource on a Specific Node

### Method 1 — nodeName (direct assignment)

Bypasses the scheduler entirely — Pod is assigned directly to the named node.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  nodeName: worker-1       # schedule directly on this node
  containers:
    - name: app
      image: nginx
```

> Limitation: If the node doesn't exist or is unavailable, the Pod stays Pending forever. Not recommended for production.

---

### Method 2 — nodeSelector (label-based)

Schedules the Pod on any node that has the matching label.

```bash
# Add a label to a node
kubectl label node worker-1 disktype=ssd
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  nodeSelector:
    disktype: ssd          # schedule on nodes with this label
  containers:
    - name: app
      image: nginx
```

---

### Method 3 — Node Affinity (advanced label-based)

More expressive than `nodeSelector` — supports `required` and `preferred` rules.

#### Required (hard rule — must match)

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: disktype
                operator: In
                values:
                  - ssd
```

#### Preferred (soft rule — try to match)

```yaml
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 1
          preference:
            matchExpressions:
              - key: disktype
                operator: In
                values:
                  - ssd
```

#### Operators available

| Operator | Meaning |
|---|---|
| `In` | Label value is in the list |
| `NotIn` | Label value is not in the list |
| `Exists` | Label key exists (any value) |
| `DoesNotExist` | Label key does not exist |
| `Gt` | Label value is greater than |
| `Lt` | Label value is less than |

---

### Method 4 — Taints and Tolerations (covered below)

---

## How to Stop Resources from Being Scheduled on a Node

### Method 1 — Cordon

Marks a node as **unschedulable** — no new Pods will be scheduled on it. Existing Pods continue running.

```bash
kubectl cordon <node-name>

# Verify
kubectl get nodes
# worker-1   Ready,SchedulingDisabled   <none>   10d   v1.28.0
```

To re-enable scheduling:

```bash
kubectl uncordon <node-name>
```

---

### Method 2 — Drain

**Evicts all Pods** from the node AND cordons it. Used before maintenance or decommissioning.

```bash
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```

| Flag | Purpose |
|---|---|
| `--ignore-daemonsets` | Skip DaemonSet-managed Pods (they are expected on every node) |
| `--delete-emptydir-data` | Allow eviction of Pods using emptyDir volumes (data will be lost) |
| `--force` | Evict Pods not managed by a controller (standalone Pods) |

After maintenance, uncordon the node:

```bash
kubectl uncordon <node-name>
```

---

### Method 3 — Taints

A taint on a node **repels** Pods from being scheduled there unless the Pod has a matching **toleration**.

#### Add a taint

```bash
kubectl taint node <node-name> key=value:Effect

# Example: prevent all Pods from scheduling on node worker-2
kubectl taint node worker-2 dedicated=gpu:NoSchedule
```

#### Taint Effects

| Effect | Behaviour |
|---|---|
| `NoSchedule` | New Pods without a matching toleration will not be scheduled |
| `PreferNoSchedule` | Kubernetes tries to avoid scheduling Pods without toleration (soft) |
| `NoExecute` | New Pods won't schedule AND existing Pods without toleration are evicted |

#### Remove a taint

```bash
kubectl taint node worker-2 dedicated=gpu:NoSchedule-
# The trailing "-" removes the taint
```

---

## Tolerations — Allowing Pods onto Tainted Nodes

A toleration on a Pod allows it to be scheduled on a tainted node.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "gpu"
      effect: "NoSchedule"
  containers:
    - name: app
      image: nvidia/cuda:11.0
```

### Toleration Operators

| Operator | Meaning |
|---|---|
| `Equal` | Key and value must match exactly |
| `Exists` | Only key must exist (any value matches) |

### Tolerate all taints (use with caution)

```yaml
tolerations:
  - operator: "Exists"
```

---

## Taints vs Cordon vs Drain

| | Cordon | Drain | Taint (NoSchedule) |
|---|---|---|---|
| **Stops new scheduling** | Yes | Yes | Yes (without toleration) |
| **Evicts existing Pods** | No | Yes | Only with NoExecute |
| **Reversible** | `uncordon` | `uncordon` | Remove taint with `-` |
| **Use case** | Temporary pause | Maintenance/decommission | Dedicated node groups |

---

## Node Labels

Labels are key-value pairs attached to nodes used for scheduling decisions.

```bash
# Add a label
kubectl label node worker-1 env=production

# Remove a label
kubectl label node worker-1 env-

# View labels
kubectl get nodes --show-labels
kubectl get nodes -l env=production
```

### Built-in node labels (set automatically)

| Label | Example Value |
|---|---|
| `kubernetes.io/hostname` | `worker-1` |
| `kubernetes.io/os` | `linux` |
| `kubernetes.io/arch` | `amd64` |
| `node.kubernetes.io/instance-type` | `t3.medium` |
| `topology.kubernetes.io/zone` | `us-east-1a` |
| `topology.kubernetes.io/region` | `us-east-1` |

---

## Complete Example — Dedicated GPU Node

```bash
# 1. Label the GPU node
kubectl label node worker-gpu accelerator=nvidia-gpu

# 2. Taint it so only GPU workloads land on it
kubectl taint node worker-gpu dedicated=gpu:NoSchedule
```

```yaml
# 3. Pod that targets the GPU node
apiVersion: v1
kind: Pod
metadata:
  name: gpu-workload
spec:
  nodeSelector:
    accelerator: nvidia-gpu
  tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "gpu"
      effect: "NoSchedule"
  containers:
    - name: gpu-app
      image: nvidia/cuda:11.0
```

---

## Useful Commands Summary

```bash
# Add node to cluster
kubeadm token create --print-join-command

# Node info
kubectl get nodes -o wide
kubectl describe node <node-name>
kubectl top nodes

# Labels
kubectl label node <node> key=value
kubectl label node <node> key-                        # remove label

# Cordon / Uncordon
kubectl cordon <node-name>
kubectl uncordon <node-name>

# Drain
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Taints
kubectl taint node <node> key=value:Effect
kubectl taint node <node> key=value:Effect-           # remove taint

# Check taints on a node
kubectl describe node <node-name> | grep Taint
```

---

## Interview Questions

### Conceptual

**Q: What are the components that run on every worker node?**
A: kubelet (manages Pods and communicates with API server), kube-proxy (manages network rules for Services), and the container runtime (containerd, CRI-O).

**Q: What is the difference between cordon and drain?**
A: Cordon marks a node as unschedulable — no new Pods are placed on it but existing Pods keep running. Drain goes further — it evicts all existing Pods from the node AND cordons it. Drain is used before maintenance or decommissioning a node.

**Q: What is the difference between a taint and a cordon?**
A: A cordon completely blocks all new scheduling on the node. A taint is more selective — it repels Pods that don't have a matching toleration, but Pods with the correct toleration can still be scheduled there. Taints are used to dedicate nodes to specific workloads.

**Q: What are the three taint effects?**
A: `NoSchedule` (new Pods without toleration won't schedule), `PreferNoSchedule` (soft version — tries to avoid), and `NoExecute` (won't schedule new Pods AND evicts existing Pods without toleration).

**Q: What is the difference between nodeSelector and nodeAffinity?**
A: `nodeSelector` is simple — it matches a single label exactly. `nodeAffinity` is more expressive — it supports operators (`In`, `NotIn`, `Exists`, etc.) and has both required (hard) and preferred (soft) rules.

**Q: What is nodeName and when should you use it?**
A: `nodeName` directly assigns a Pod to a specific node, bypassing the scheduler. It should rarely be used in production because if the node is unavailable, the Pod stays Pending with no fallback.

**Q: What happens to Pods on a drained node after you uncordon it?**
A: The node becomes schedulable again, but previously evicted Pods are not automatically moved back. They will be rescheduled by their controllers (Deployments, ReplicaSets) onto available nodes, which may or may not be the original node.

**Q: How do you add a new worker node to a kubeadm cluster?**
A: Run `kubeadm token create --print-join-command` on the control plane, then run the output join command on the new node after installing kubelet, kubeadm, and a container runtime.

**Q: What flag is required when draining a node that has DaemonSet-managed Pods?**
A: `--ignore-daemonsets`. Without it, drain will refuse to proceed because DaemonSet Pods cannot be evicted (they are expected to run on every node).

**Q: Can a Pod with no tolerations be scheduled on a tainted node?**
A: No. A Pod without a matching toleration will not be scheduled on a node with a `NoSchedule` or `NoExecute` taint. It may be scheduled on a node with `PreferNoSchedule` if no other nodes are available.

### Scenario-Based

**Q: You need to take worker-1 offline for OS patching. How do you do it safely?**
A:
```bash
# Evict all Pods and cordon the node
kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data

# Perform OS maintenance...

# Bring node back into rotation
kubectl uncordon worker-1
```

**Q: You want only GPU workloads to run on node worker-gpu and prevent all other Pods from landing on it. How?**
A:
```bash
# Taint the node
kubectl taint node worker-gpu dedicated=gpu:NoSchedule
```
Then add a matching toleration to GPU Pods. All other Pods without the toleration will not be scheduled there.

**Q: A Pod is stuck in Pending. You suspect it's a node scheduling issue. How do you debug?**
A:
```bash
# Check Pod events for scheduling errors
kubectl describe pod <pod-name>

# Check node status and taints
kubectl get nodes
kubectl describe node <node-name> | grep -E "Taint|Condition|Pressure"

# Check if nodeSelector or affinity rules match any node
kubectl get nodes --show-labels
```

**Q: You want a Pod to prefer running on nodes in zone `us-east-1a` but still run elsewhere if unavailable. How?**
A: Use `preferredDuringSchedulingIgnoredDuringExecution` node affinity:
```yaml
affinity:
  nodeAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
            - key: topology.kubernetes.io/zone
              operator: In
              values:
                - us-east-1a
```

**Q: How do you check which taints are on a node?**
A:
```bash
kubectl describe node <node-name> | grep Taint
# or
kubectl get node <node-name> -o jsonpath='{.spec.taints}'
```

---

## Summary

| Topic | Key Command / Concept |
|---|---|
| Add node to cluster | `kubeadm token create --print-join-command` |
| Schedule on specific node | `nodeName`, `nodeSelector`, `nodeAffinity` |
| Block all scheduling | `kubectl cordon <node>` |
| Evict Pods + block scheduling | `kubectl drain <node> --ignore-daemonsets` |
| Repel Pods selectively | `kubectl taint node <node> key=value:Effect` |
| Allow Pod onto tainted node | Add `tolerations` to Pod spec |
| Re-enable scheduling | `kubectl uncordon <node>` |
