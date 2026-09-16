# KRB Academy — Kubernetes Concepts
## Session Date: 2026-09-16

## Topics Covered

### 1. Pod Lifecycle and Graceful Termination

Kubernetes Pods have a lifecycle that includes creation, running, termination, and removal.

When Kubernetes terminates a Pod, the intended behavior is graceful termination rather than an immediate forced kill.

```text
Pod selected for termination
        ↓
Pod removed from Service endpoints
        ↓
SIGTERM sent to container process
        ↓
Application performs graceful shutdown
        ↓
terminationGracePeriodSeconds expires
        ↓
SIGKILL if process is still running
        ↓
Pod removed
```

### 2. Readiness During Pod Termination

Readiness is important during shutdown because a terminating Pod should stop receiving new traffic.

A Pod that becomes unready is removed from the endpoints used by its Service.

This creates an important distinction:

- **Readiness** — should this Pod receive traffic?
- **Liveness** — should Kubernetes restart this container?
- **Startup** — has the application finished starting?

### 3. SIGTERM and Graceful Shutdown

Kubernetes sends `SIGTERM` when a Pod is being terminated.

A well-behaved application should use the shutdown window to:

- stop accepting new work
- allow in-flight requests to complete
- close connections and resources
- stop background processing cleanly

If the process is still running after the grace period, Kubernetes can force termination with `SIGKILL`.

### 4. terminationGracePeriodSeconds

`terminationGracePeriodSeconds` defines how long Kubernetes gives a Pod to terminate gracefully.

Example:

```yaml
spec:
  terminationGracePeriodSeconds: 30
```

The value should reflect the application's actual shutdown requirements.

### 5. Rolling Updates and Readiness

Rolling updates depend heavily on readiness.

```text
Old Pod(s)
   ↓
New Pod created
   ↓
New Pod becomes Ready
   ↓
New Pod can receive traffic
   ↓
Old Pod is terminated gracefully
```

Readiness therefore acts as a traffic-safety mechanism during deployments.

### 6. Startup vs Readiness vs Liveness

| Probe | Main Question | Failure Effect |
|---|---|---|
| Startup | Has the application started successfully? | Prevents liveness/readiness from acting too early |
| Readiness | Can this Pod receive traffic? | Pod is removed from Service endpoints |
| Liveness | Is the running container still healthy? | Container may be restarted |

A common production mistake is using liveness to represent readiness.

An application can be alive but temporarily unable to serve traffic because it is warming caches, initializing, waiting for a dependency, or overloaded. In such cases, readiness can fail without necessarily requiring a restart.

## Interview Takeaways

1. Readiness protects traffic; liveness protects process health.
2. SIGTERM gives the application an opportunity to shut down cleanly.
3. `terminationGracePeriodSeconds` defines the graceful shutdown window.
4. Rolling deployments depend on readiness to avoid routing traffic to unready Pods.
5. Startup probes protect slow-starting applications from premature liveness failures.
6. Kubernetes lifecycle management and application graceful shutdown must be designed together.

## Key Mental Model

```text
Deployment
    ↓
Pod lifecycle
    ↓
Readiness controls traffic
    ↓
SIGTERM starts graceful shutdown
    ↓
terminationGracePeriodSeconds
    ↓
SIGKILL only if necessary
```

## Next Topic

**PodDisruptionBudget (PDB)**
