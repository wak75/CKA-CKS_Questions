# Multiple Containers in One Pod

## What is a Multi-Container Pod?

A Pod is the smallest deployable unit in Kubernetes. While a Pod typically runs a single container, Kubernetes allows multiple containers to coexist in the same Pod. These containers share the same network namespace (same IP, same localhost) and the same storage volumes.

---

## Why Use Multiple Containers in a Pod?

Containers in the same Pod are tightly coupled — they live and die together, scale together, and are scheduled on the same node. Use this pattern only when containers are genuinely interdependent, not just related.

Common reasons:
- A helper container needs to run alongside the main app
- Shared data/files between processes (via shared volumes)
- Network proximity matters (localhost communication)

---

## Multi-Container Design Patterns

### 1. Sidecar Pattern
A helper container that enhances or extends the main container.

**Examples:**
- Log shipper (e.g., Fluentd) reading logs from the main app
- Config reloader watching for config changes
- Service mesh proxy (e.g., Envoy in Istio)

### 2. Ambassador Pattern
A proxy container that handles external communication on behalf of the main container.

**Examples:**
- A container that proxies database connections
- Rate-limiting proxy in front of an API

### 3. Adapter Pattern
Transforms or normalizes output from the main container for external systems.

**Examples:**
- Converting app metrics to Prometheus format
- Normalizing log formats before forwarding

---

## How to Create a Multi-Container Pod

### Basic Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
  labels:
    app: demo
spec:
  containers:
    - name: main-app
      image: nginx:latest
      ports:
        - containerPort: 80
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/nginx

    - name: log-sidecar
      image: busybox
      command: ["sh", "-c", "tail -f /var/log/nginx/access.log"]
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/nginx

  volumes:
    - name: shared-logs
      emptyDir: {}
```

### Sidecar with Init Container Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-pod
spec:
  initContainers:
    - name: init-setup
      image: busybox
      command: ["sh", "-c", "echo 'init done' > /work/status.txt"]
      volumeMounts:
        - name: work-dir
          mountPath: /work

  containers:
    - name: main-app
      image: nginx:latest
      volumeMounts:
        - name: work-dir
          mountPath: /usr/share/nginx/html

    - name: sidecar
      image: busybox
      command: ["sh", "-c", "while true; do echo sidecar running; sleep 10; done"]

  volumes:
    - name: work-dir
      emptyDir: {}
```

---

## Key Characteristics

| Property | Detail |
|---|---|
| **Shared Network** | All containers share the same IP and port space; communicate via `localhost` |
| **Shared Volumes** | Volumes defined in the Pod spec can be mounted by any container |
| **Lifecycle** | All containers start/stop together (Pod lifecycle) |
| **Scheduling** | Always scheduled on the same node |
| **Resource limits** | Each container has its own CPU/memory limits |

---

## Container Communication

Since all containers share the same network namespace:

```bash
# Container A can reach Container B via localhost
curl http://localhost:8080
```

They can also share data via a mounted `emptyDir` volume.

---

## Init Containers vs Sidecar Containers

| | Init Container | Sidecar Container |
|---|---|---|
| **When it runs** | Before main containers start | Alongside main containers |
| **Must complete?** | Yes — must exit 0 before Pod proceeds | No — runs continuously |
| **Use case** | Setup, pre-checks, migrations | Logging, proxying, monitoring |
| **Restart** | Re-runs if Pod restarts | Restarts with the Pod |

---

## Resource Configuration per Container

Each container in a Pod can have its own resource requests and limits:

```yaml
containers:
  - name: main-app
    image: nginx
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"

  - name: sidecar
    image: busybox
    resources:
      requests:
        memory: "32Mi"
        cpu: "100m"
      limits:
        memory: "64Mi"
        cpu: "200m"
```

---

## Useful kubectl Commands

```bash
# List all containers in a pod
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[*].name}'

# Exec into a specific container
kubectl exec -it <pod-name> -c <container-name> -- /bin/sh

# View logs of a specific container
kubectl logs <pod-name> -c <container-name>

# View logs of all containers
kubectl logs <pod-name> --all-containers=true

# Describe pod (shows all containers and their states)
kubectl describe pod <pod-name>
```

---

## Interview Questions

### Conceptual

**Q: What is a multi-container Pod and when would you use it?**
A: A multi-container Pod runs more than one container in the same Pod, sharing the same network and storage. Use it when containers are tightly coupled — e.g., a log shipper that must read files from the main app, or a proxy that intercepts all traffic.

**Q: How do containers in the same Pod communicate?**
A: They share the same network namespace, so they communicate over `localhost` using port numbers. They can also share data via mounted volumes.

**Q: What is the Sidecar pattern?**
A: A secondary container that runs alongside the main container to extend or enhance its functionality without changing the main container's code. Examples: log forwarders, config reloaders, service mesh proxies.

**Q: What is the difference between an Init Container and a Sidecar?**
A: Init containers run sequentially before the main containers start and must complete successfully. Sidecars run concurrently with the main container for the lifetime of the Pod.

**Q: Do containers in a Pod share the same filesystem?**
A: No, each container has its own filesystem. To share data, you must explicitly define a volume in the Pod spec and mount it in each container that needs access.

**Q: What happens if one container in a multi-container Pod crashes?**
A: Only that container restarts (based on the Pod's `restartPolicy`). The other containers continue running. If the Pod's restart policy is `Never`, the failed container stays in a terminated state.

**Q: Can two containers in the same Pod listen on the same port?**
A: No. Since they share the same network namespace, each container must listen on a different port — just like two processes on the same machine.

### Scenario-Based

**Q: You have a web app that generates logs to a file. You want to ship those logs to an external system without modifying the app. How would you design this?**
A: Use a sidecar container (e.g., Fluentd or Filebeat) that mounts the same `emptyDir` volume as the main app. The main app writes logs to the shared volume; the sidecar reads and forwards them.

**Q: When would you NOT use a multi-container Pod?**
A: When the containers are loosely coupled and can scale independently. In that case, use separate Pods (and Services to communicate). Multi-container Pods should be reserved for tight coupling only.

**Q: How would you debug a failing sidecar in a multi-container Pod?**
A:
```bash
kubectl describe pod <pod-name>        # check events and container states
kubectl logs <pod-name> -c <sidecar>   # view sidecar logs
kubectl exec -it <pod-name> -c <sidecar> -- /bin/sh  # exec in if running
```

---

## Common Pitfalls

- **Port conflicts**: Two containers cannot bind to the same port since they share the network namespace.
- **Over-coupling**: Putting loosely related containers in one Pod removes independent scalability.
- **Resource starvation**: If one container consumes too many resources, it can affect the other. Always set resource limits.
- **Startup ordering**: Regular sidecar containers don't wait for each other. Use init containers for strict ordering.

---

## Summary

| Concept | Key Point |
|---|---|
| Shared network | Same IP, use `localhost` to communicate |
| Shared storage | Use `emptyDir` or PVC volumes mounted in multiple containers |
| Patterns | Sidecar, Ambassador, Adapter |
| Init vs Sidecar | Init runs before; sidecar runs alongside |
| Debugging | `kubectl logs -c`, `kubectl exec -c`, `kubectl describe pod` |
| When to use | Tight coupling only — log shipping, proxying, config watching |
