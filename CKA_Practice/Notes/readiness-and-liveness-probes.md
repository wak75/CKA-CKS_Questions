# Readiness and Liveness Probes

## What are Probes?

Probes are periodic diagnostic checks that the **kubelet** runs on a container to determine its health and readiness. Kubernetes uses the results to make decisions — whether to send traffic to the Pod, or restart the container.

There are three types of probes:
| Probe | Purpose |
|---|---|
| **Liveness** | Is the container still running correctly? |
| **Readiness** | Is the container ready to serve traffic? |
| **Startup** | Has the container finished starting up? |

---

## Liveness Probe

### What it does
Checks if the container is **alive and healthy**. If the probe fails, the kubelet **kills and restarts** the container based on the Pod's `restartPolicy`.

### When to use
- When your app can get into a broken/deadlocked state but the process is still running
- When a hung thread or memory leak causes the app to become unresponsive without crashing
- Long-running background workers or processing jobs

### When NOT to use
- Do not set `initialDelaySeconds` too low — the container may not have started yet and will be killed prematurely
- Avoid using it if your app crashes on its own (Kubernetes already handles that)

### Example
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 20
  timeoutSeconds: 2
  failureThreshold: 3
```

---

## Readiness Probe

### What it does
Checks if the container is **ready to accept traffic**. If the probe fails, the Pod is removed from the Service's endpoints — no traffic is routed to it. The container is **not restarted**.

### When to use
- When your app takes time to warm up (loading caches, DB connections, config)
- When your app temporarily becomes unable to serve traffic (e.g., during a batch job or high load)
- Deployment rollouts — ensures new Pods are healthy before old ones are removed

### When NOT to use
- Do not use it as a substitute for liveness — readiness failure never restarts the container
- Avoid overly strict checks that cause Pods to flap in/out of service

### Example
```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
  timeoutSeconds: 2
  failureThreshold: 3
```

---

## Startup Probe

### What it does
Gives a **slow-starting container** time to initialize before liveness/readiness probes kick in. Once the startup probe succeeds, it hands off to liveness and readiness probes.

### When to use
- Legacy apps or JVM apps that take a long time to start
- Prevents liveness probe from killing the container during startup

### Example
```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
  # allows up to 30 * 10 = 300 seconds to start
```

---

## Probe Types (How Checks Are Performed)

### 1. HTTP GET
Makes an HTTP GET request. Success = status code 200–399.

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
    httpHeaders:
      - name: Custom-Header
        value: health-check
```

### 2. TCP Socket
Opens a TCP connection to the specified port. Success = connection established.

```yaml
livenessProbe:
  tcpSocket:
    port: 3306
```

### 3. Exec Command
Runs a command inside the container. Success = exit code 0.

```yaml
livenessProbe:
  exec:
    command:
      - cat
      - /tmp/healthy
```

### 4. gRPC (Kubernetes 1.24+)
Calls a gRPC health check endpoint.

```yaml
livenessProbe:
  grpc:
    port: 2379
```

---

## Probe Configuration Fields

| Field | Description | Default |
|---|---|---|
| `initialDelaySeconds` | Seconds to wait before first probe after container starts | 0 |
| `periodSeconds` | How often to run the probe | 10 |
| `timeoutSeconds` | Seconds before probe times out | 1 |
| `successThreshold` | Min consecutive successes to mark healthy | 1 |
| `failureThreshold` | Min consecutive failures before action is taken | 3 |

---

## Liveness vs Readiness vs Startup — Side by Side

| | Liveness | Readiness | Startup |
|---|---|---|---|
| **Purpose** | Is it alive? | Is it ready for traffic? | Has it finished starting? |
| **On failure** | Restart container | Remove from Service endpoints | Restart container |
| **On success** | Do nothing | Add to Service endpoints | Hand off to liveness/readiness |
| **Runs** | Throughout lifetime | Throughout lifetime | Only during startup |
| **Replaces** | — | — | liveness during startup |

---

## Complete Example: All Three Probes

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: full-probe-example
spec:
  containers:
    - name: app
      image: myapp:1.0
      ports:
        - containerPort: 8080

      startupProbe:
        httpGet:
          path: /healthz
          port: 8080
        failureThreshold: 30
        periodSeconds: 10

      livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
        initialDelaySeconds: 0
        periodSeconds: 20
        timeoutSeconds: 2
        failureThreshold: 3

      readinessProbe:
        httpGet:
          path: /ready
          port: 8080
        initialDelaySeconds: 0
        periodSeconds: 10
        timeoutSeconds: 2
        failureThreshold: 3
```

> Note: When a `startupProbe` is defined, liveness and readiness probes do not start until it succeeds. So `initialDelaySeconds` on liveness/readiness can be set to 0.

---

## Where to Implement Probes

| Scenario | Probe to Use |
|---|---|
| Web servers / REST APIs | HTTP GET on `/healthz` or `/ready` |
| Databases (MySQL, Redis) | TCP socket on the DB port |
| Background workers | Exec command checking a lock/status file |
| gRPC services | gRPC health check |
| Slow-starting JVM / legacy apps | Startup probe first, then liveness |
| Deployment rollouts | Readiness probe (gates traffic switch) |

---

## Benefits

### Liveness Probe
- Automatically recovers from deadlocks or frozen processes
- No manual intervention needed when app gets stuck
- Improves overall system reliability

### Readiness Probe
- Prevents traffic from reaching unready Pods during startup
- Enables zero-downtime deployments (new Pod must be ready before old one is removed)
- Handles temporary unavailability gracefully (e.g., during GC pauses or batch jobs)

### Startup Probe
- Safely handles slow-starting containers without aggressive liveness restarts
- Replaces the hacky `initialDelaySeconds` workaround for long startups

---

## Common Mistakes

| Mistake | Problem | Fix |
|---|---|---|
| No `initialDelaySeconds` on liveness | Container killed before it starts | Set a reasonable delay or use `startupProbe` |
| Same endpoint for liveness and readiness | A temporary overload kills the container | Use different endpoints — `/healthz` for liveness, `/ready` for readiness |
| `failureThreshold: 1` on liveness | One transient failure causes restart | Set to 3+ to tolerate brief hiccups |
| Very low `timeoutSeconds` | Slow but healthy app gets false failures | Set based on your app's p99 response time |
| Readiness probe too strict | Pod constantly flaps in/out of service | Loosen the check or increase `failureThreshold` |
| Using exec probe with slow commands | Command timeout causes false failure | Ensure the command exits quickly |

---

## Interview Questions

### Conceptual

**Q: What is the difference between a Liveness and Readiness probe?**
A: Liveness checks if the container is healthy — failure causes a restart. Readiness checks if the container can serve traffic — failure removes the Pod from Service endpoints without restarting it.

**Q: What happens when a Readiness probe fails?**
A: The Pod is removed from the endpoints of any Service that selects it. Traffic stops being routed to it, but the container is not restarted. Once the probe passes again, the Pod is re-added to endpoints.

**Q: What happens when a Liveness probe fails?**
A: The kubelet kills the container and restarts it according to the Pod's `restartPolicy` (default is `Always`).

**Q: What is a Startup probe and why is it useful?**
A: A startup probe gates the liveness and readiness probes until the container finishes initializing. It is useful for slow-starting apps (e.g., JVM apps) where a liveness probe would kill the container before it's ready.

**Q: Can you use only a Readiness probe without a Liveness probe?**
A: Yes. If your app crashes on its own, Kubernetes already restarts it. In that case, a readiness probe alone is sufficient to manage traffic routing.

**Q: What are the three probe mechanisms?**
A: HTTP GET (checks an HTTP endpoint), TCP Socket (checks if a port is open), and Exec (runs a command inside the container and checks the exit code).

**Q: What is `failureThreshold` and why does it matter?**
A: It is the number of consecutive probe failures required before Kubernetes takes action. It prevents flapping — a single transient failure doesn't immediately kill or remove the container.

**Q: If both liveness and readiness probes are configured, in what order do they apply?**
A: Both run concurrently throughout the Pod's lifetime. If a `startupProbe` is also defined, neither liveness nor readiness runs until the startup probe succeeds.

**Q: What is the default behavior if no probes are configured?**
A: Kubernetes assumes the container is always alive and always ready as soon as it starts. There are no health checks.

### Scenario-Based

**Q: Your app loads a large cache at startup and takes 60 seconds. The liveness probe is killing it before it's ready. How do you fix this?**
A: Add a `startupProbe` with a `failureThreshold` and `periodSeconds` that together allow enough time (e.g., `failureThreshold: 12`, `periodSeconds: 10` = 120 seconds). This prevents liveness from running until startup succeeds.

**Q: During a rolling deployment, new Pods receive traffic before they are ready, causing errors. What is missing?**
A: A `readinessProbe`. Without it, Kubernetes adds Pods to Service endpoints as soon as they start, regardless of whether the app is ready. Adding a readiness probe ensures traffic only reaches healthy Pods.

**Q: Your app sometimes gets into a deadlock state — the process is running but it stops responding. Which probe solves this?**
A: A `livenessProbe`. It detects that the app is no longer responding (HTTP timeout or exec command fails) and restarts the container automatically.

**Q: How would you verify that your probes are working?**
A:
```bash
kubectl describe pod <pod-name>   # Shows probe config and recent events
kubectl get events                 # Shows probe failure events
kubectl logs <pod-name>            # Check app logs around restart times
```

**Q: You have a batch-processing container that should not receive HTTP traffic while it is processing a job. How do you handle this?**
A: Implement a readiness probe that checks a status file or endpoint. The app sets itself to "not ready" at the start of a job and "ready" when done. Kubernetes will remove it from Service endpoints during processing.

---

## Summary

| | Liveness | Readiness | Startup |
|---|---|---|---|
| **Failure action** | Restart container | Remove from traffic | Restart container |
| **Use for** | Deadlocks, frozen apps | Slow startup, temp unavailability | Slow-starting apps |
| **Probe types** | HTTP, TCP, Exec, gRPC | HTTP, TCP, Exec, gRPC | HTTP, TCP, Exec, gRPC |
| **Key field** | `failureThreshold` | `initialDelaySeconds` | `failureThreshold` |
| **Impact on Service** | No | Yes | No |
