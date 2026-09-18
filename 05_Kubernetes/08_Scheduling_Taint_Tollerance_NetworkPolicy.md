# KRB Academy — Kubernetes Concepts
## Session Date: 2026-09-18

## Scheduling
- The Kubernetes Scheduler selects a suitable node for an unscheduled Pod.
- CPU and memory **requests** are primary scheduling inputs; limits are runtime constraints.
- A Pod whose request cannot fit remains `Pending`.
- Example: node allocatable CPU = 8, Pod request = 20 CPU → `Insufficient cpu`.
- Preemption can remove lower-priority Pods only when that can make scheduling possible. It cannot solve an impossible request such as 20 CPU on an 8 CPU node.

## nodeSelector
`nodeSelector` is a hard node-label requirement:
```yaml
nodeSelector:
  kubernetes.io/os: linux
```
The Pod can only schedule onto nodes matching the label. A mismatch leaves the Pod Pending.

## Node Affinity
- `requiredDuringSchedulingIgnoredDuringExecution` = hard requirement.
- `preferredDuringSchedulingIgnoredDuringExecution` = preference.
- `weight` expresses preference strength.
Mental model:
```text
Hard requirements → filter eligible nodes
Preferences → score eligible nodes
```

## Taints and Tolerations
- Taint = restriction placed on a node.
- Toleration = allows a Pod to be considered for a matching tainted node.
- A toleration does not force placement.
Effects:
- `NoSchedule` → do not schedule new non-tolerating Pods.
- `PreferNoSchedule` → avoid them where possible.
- `NoExecute` → do not schedule and may evict existing non-tolerating Pods.

## Pod Priority and Preemption
Higher-priority Pods may preempt lower-priority Pods when that makes scheduling possible. Preemption is a scheduler mechanism; eviction is a separate mechanism.

## Pod Anti-Affinity
Anti-affinity can distribute replicas across nodes, reducing the chance that one node failure removes all replicas. `topologyKey: kubernetes.io/hostname` can express node-level distribution.

## SecurityContext
Controls security-related execution settings:
```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
```
Other hardening can include `readOnlyRootFilesystem: true`.

## RBAC and ServiceAccounts
RBAC controls who can perform operations against the Kubernetes API.
Core objects: Role, RoleBinding, ClusterRole, ClusterRoleBinding.
Conceptual flow:
```text
ServiceAccount → RoleBinding → Role → Kubernetes API permissions
```
Role is namespace-scoped. Kubernetes RBAC is separate from Spring Security authorization inside KRB Enterprise.

## NetworkPolicy
NetworkPolicy controls allowed network traffic between Pods/endpoints, subject to CNI enforcement.
KRB Enterprise model:
```text
Ingress → KRB Enterprise
KRB Enterprise → PostgreSQL
Other Pods → PostgreSQL restricted
```
Service provides stable networking/Pod selection; NetworkPolicy controls network authorization.

## Jobs and CronJobs
- Job runs work until completion.
- CronJob creates Jobs on a schedule.
```text
CronJob → Job → Pod
```

## Deployment vs StatefulSet
- Deployment: interchangeable/stateless application replicas.
- StatefulSet: stable identity and/or persistent storage association, e.g. `postgres-0`.

## Kubernetes Architecture
```text
Control Plane
  API Server
  Scheduler
  Controller Manager
  etcd

Worker Node
  kubelet
  kube-proxy
  container runtime
```
- API Server = Kubernetes API entry point.
- etcd = cluster state.
- Scheduler = Pod placement.
- Controller Manager = reconciliation.
- kubelet = ensures assigned Pods run.
- Container runtime = runs containers.

## Desired State and Reconciliation
Kubernetes is declarative. Controllers continuously move actual state toward desired state.
Example:
```text
Desired replicas = 3
Actual replicas = 2
→ controller creates a Pod
→ actual = 3
```

## Deployment Strategies
Common strategies:
- Rolling update
- Blue/Green
- Canary
- Recreate

## Observability
Three common pillars:
- Metrics
- Logs
- Traces

KRB Enterprise uses Metrics Server / `kubectl top`, Spring Boot Actuator health, and has Dynatrace/Grafana exposure.
Average CPU is not the same as p95/p99 latency.

## Troubleshooting
Pending:
```bash
kubectl describe pod <pod>
```
Check scheduler Events for insufficient resources, selector/affinity mismatch, taints, storage constraints, etc.

CrashLoopBackOff:
```bash
kubectl logs <pod>
kubectl logs <pod> --previous
kubectl describe pod <pod>
```

Service traffic path:
```text
Service → selector → endpoints → readiness → Pod → container port
```

## KRB Enterprise Mental Model
```text
kubectl apply
→ API Server
→ etcd
→ Controller
→ ReplicaSet
→ Scheduler
→ kubelet
→ Ready Pod
→ Service
→ Ingress
→ users
```

Scaling:
```text
Metrics → HPA → Deployment replicas → Pods → Scheduler → Nodes
```

Availability:
```text
Deployment → replicas
PDB → voluntary disruption protection
Scheduler → placement
Anti-affinity → distribution
Probes → health/traffic eligibility
```

## Status
Kubernetes concepts and core hands-on learning are complete for the current milestone. GitHub Actions → Kubernetes deployment integration remains for the next session.
