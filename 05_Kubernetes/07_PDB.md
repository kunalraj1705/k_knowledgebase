# KRB Academy — Kubernetes PDB Notes
## Session Date: 2026-09-17

## Topics Covered

### 1. PodDisruptionBudget (PDB)

A PodDisruptionBudget protects application availability during **voluntary disruptions**.

The key question a PDB answers is:

> How many Pods can Kubernetes voluntarily disrupt while maintaining the application's availability requirement?

PDBs do not control normal HPA scaling and do not protect against every possible failure.

---

## 2. Voluntary vs Involuntary Disruptions

### Voluntary disruptions

Examples:

- Node drain
- Planned node maintenance
- Cluster maintenance
- Administrative Pod eviction
- Certain planned infrastructure operations

### Involuntary disruptions

Examples:

- Node crash
- Hardware failure
- Kernel failure
- Unexpected container/process failure

A PDB primarily protects against **voluntary disruptions**. It is not a guarantee that Pods will always remain available.

---

## 3. `minAvailable`

`minAvailable` specifies the minimum number of Pods that should remain available during voluntary disruption.

Example:

```yaml
spec:
  minAvailable: 2
```

If 5 Pods are available:

```text
5 available
   ↓
evict 1 → 4 available
evict 1 → 3 available
evict 1 → 2 available
evict 1 → 1 available ❌
```

The fourth eviction would violate the PDB.

---

## 4. `maxUnavailable`

`maxUnavailable` specifies the maximum number of Pods that may be unavailable because of voluntary disruption.

Example:

```yaml
spec:
  maxUnavailable: 1
```

With 5 available Pods, the disruption budget permits at most one Pod to be unavailable under the budget.

---

## 5. Absolute Values vs Percentages

PDB availability can be expressed using either an absolute number or a percentage.

Absolute:

```yaml
minAvailable: 2
```

Percentage:

```yaml
minAvailable: 50%
```

Percentage-based policies adapt as the number of Pods changes, while an absolute value such as `minAvailable: 2` continues to require two available Pods.

---

## 6. PDB Does Not Scale Pods

A critical distinction:

```text
HPA
→ Determines desired replica count from metrics

Deployment
→ Maintains the desired number of Pods

Scheduler
→ Determines where Pods run

PDB
→ Limits voluntary disruption
```

A PDB does **not** create replacement Pods.

If a Pod is evicted and the Deployment still requires that replica, the **Deployment controller** ensures the desired replica count is restored.

---

## 7. PDB and HPA

KRB Enterprise uses:

```text
HPA minReplicas = 2
HPA maxReplicas = 5
```

With:

```yaml
minAvailable: 2
```

the two settings have different responsibilities.

### HPA

Defines the normal scaling range:

```text
2 → 5 replicas
```

### PDB

Protects against voluntary disruption:

```text
At least 2 available Pods
```

When HPA scales the workload down, the PDB is not the mechanism defining the scaling floor.

---

## 8. PDB and Rolling Updates

PDB and Deployment rolling-update behavior solve different problems.

```text
RollingUpdate
→ Controls how a new application version replaces the old version

PDB
→ Protects availability during voluntary disruption
```

Readiness is also important because a new Pod should become Ready before receiving normal Service traffic.

---

## 9. PDB and Node Drain

A node drain uses the Kubernetes eviction mechanism for Pods that can be voluntarily disrupted.

The PDB is consulted during eviction.

Example:

```text
4 available Pods
PDB minAvailable = 2

Eviction 1 → allowed
Available = 3

Eviction 2 → allowed
Available = 2

Eviction 3 → blocked
Available would become 1
```

The Kubernetes eviction operation can therefore return an error similar to:

```text
Cannot evict pod as it would violate the pod's disruption budget.
```

---

## 10. PDB Does Not Move Pods

A PDB only controls whether a voluntary disruption can proceed.

It does not decide where replacement Pods run.

That responsibility belongs to the scheduler.

```text
PDB
→ Can this Pod be voluntarily disrupted?

Deployment
→ Do I still need this replica?

Scheduler
→ Where can the replacement Pod run?
```

---

## 11. Single-Node Cluster Limitation

A single-node cluster cannot demonstrate true high-availability node maintenance.

If all application Pods are on one node:

```text
Node A
├── Pod 1
├── Pod 2
├── Pod 3
└── Pod 4
```

there is nowhere else for replacement Pods to run if Node A is unavailable or cordoned.

Production Kubernetes environments normally use multiple worker nodes when workload availability during node maintenance is required.

---

## Interview Takeaways

1. PDB protects against **voluntary disruptions**.
2. `minAvailable` defines the minimum available Pods.
3. `maxUnavailable` defines the maximum unavailable Pods.
4. PDB does not control HPA scaling.
5. PDB does not create replacement Pods.
6. Deployment maintains desired replicas.
7. Scheduler determines Pod placement.
8. PDB is evaluated during Kubernetes eviction operations such as node drain.
9. PDB does not protect against involuntary failures such as a node crash.
10. A single-node cluster limits what can be demonstrated about Pod rescheduling and high availability.

## Key Mental Model

```text
                         HPA
                          │
                  Replica calculation
                          ↓
                     Deployment
                          │
                  Maintains replicas
                          ↓
                        Pods
                          │
                     Scheduler
                          │
                  Pod placement
                          ↑
                          │
                         PDB
             Voluntary disruption guard
```

## Next Topic

**Kubernetes Scheduling**

Planned topics:

- CPU/memory requests and scheduling
- Node capacity
- Taints and tolerations
- Node affinity
- Pod affinity / anti-affinity
- Topology spread constraints
