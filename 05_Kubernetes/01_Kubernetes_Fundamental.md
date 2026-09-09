# KRB Academy — Kubernetes Notes
## Session: Kubernetes Fundamentals & KRB Enterprise Architecture
**Date:** 2026-09-09

---

## 1. Topics Covered

- Kubernetes Cluster
- Namespaces
- Pods
- Deployments
- ReplicaSets
- Services
- Kubernetes DNS / service discovery
- StatefulSets
- Persistent storage
- ConfigMaps
- Secrets
- Startup, liveness, and readiness probes
- Rolling updates
- Stateless vs stateful workloads
- Application-to-database communication

---

## 2. KRB Enterprise UAT Architecture

Current example:

### Core Services
1. `core-krb-payment`
2. `core-krb-customer`
3. `core-krb-order`

### Non-Core Services
4. `non-krb-application`
5. `non-krb-partner`

### UI
6. `krb-application-ui`
7. `krb-partner-ui`

Current database model:

`PostgreSQL → krb_enterprise`

Conceptually:

```text
UAT Environment
│
├── Kubernetes Cluster
│   ├── uat-core
│   │   ├── core-krb-payment
│   │   ├── core-krb-customer
│   │   └── core-krb-order
│   ├── uat-noncore
│   │   ├── non-krb-application
│   │   └── non-krb-partner
│   └── uat-ui
│       ├── krb-application-ui
│       └── krb-partner-ui
│
└── PostgreSQL
    └── krb_enterprise
```

---

## 3. Cluster vs Namespace

A **cluster** is the complete Kubernetes environment containing the control plane and worker nodes.

A **namespace** is a logical separation inside a cluster.

```text
UAT Cluster
├── uat-core
├── uat-noncore
└── uat-ui
```

Important:

> A namespace does not create a separate cluster or worker node. Different namespaces can run Pods on the same worker nodes.

---

## 4. Core Kubernetes Hierarchy

For a typical stateless Spring Boot microservice:

```text
Cluster
  ↓
Namespace
  ↓
Deployment
  ↓
ReplicaSet
  ↓
Pod
  ↓
Container
```

Networking:

```text
Service
  ↓
Pods
```

---

## 5. Deployment

A Deployment represents the desired state of a stateless application and manages application rollouts.

```text
Deployment
    ↓
ReplicaSet
    ↓
Pods
```

If desired replicas = 3, Kubernetes continuously works toward maintaining three Pods.

A Deployment also manages version updates and rollbacks.

---

## 6. ReplicaSet

A ReplicaSet maintains the desired number of matching Pods.

```text
ReplicaSet
├── Pod A
├── Pod B
└── Pod C
```

If Pod B disappears:

```text
ReplicaSet
├── Pod A
├── Pod C
└── Pod D  ← replacement
```

Normally, the Deployment manages ReplicaSets rather than engineers managing them directly.

---

## 7. Pod

A Pod is Kubernetes' smallest deployable unit.

For a simple Spring Boot service:

```text
Pod
└── Spring Boot container
```

Pods have network identities/IPs, but Pod IPs are temporary.

```text
Old Pod
10.244.2.15
   ↓
deleted

New Pod
10.244.3.27
```

Therefore applications should normally communicate through Services rather than directly through Pod IPs.

---

## 8. Service

A Service provides a stable network endpoint for a group of Pods.

```text
core-krb-payment-service
          │
     ┌────┼────┐
     ↓    ↓    ↓
   Pod A Pod B Pod C
```

Other services communicate with the Service rather than individual Pod IPs.

Example:

```text
core-krb-order
       ↓
core-krb-payment-service:8080
       ↓
Payment Pods
```

---

## 9. Kubernetes DNS

Kubernetes provides DNS-based service discovery.

Instead of:

```text
http://10.244.2.15:8080
```

use the Service name:

```text
http://core-krb-payment-service:8080
```

For cross-namespace communication:

```text
http://core-krb-payment-service.uat-core:8080
```

The application does not need to know changing Pod IP addresses.

---

## 10. Rolling Updates

Suppose:

```text
core-krb-payment:v1
```

has three Pods.

Deploying:

```text
core-krb-payment:v2
```

causes the Deployment to create a new ReplicaSet and gradually replace the old Pods.

Conceptually:

```text
Deployment
├── ReplicaSet v1
│   ├── Pod v1
│   └── Pod v1
└── ReplicaSet v2
    ├── Pod v2
    └── Pod v2
```

Eventually:

```text
Deployment
└── ReplicaSet v2
    ├── Pod v2
    ├── Pod v2
    └── Pod v2
```

The exact overlap depends on the configured rollout strategy.

---

## 11. Deployment Isolation

If only `core-krb-payment` is updated, Kubernetes does not automatically redeploy the other workloads.

Unchanged:

```text
core-krb-customer
core-krb-order
non-krb-application
non-krb-partner
krb-application-ui
krb-partner-ui
PostgreSQL
```

Each application normally has its own Deployment.

---

## 12. Scheduler and Worker Nodes

Pods run on worker nodes.

```text
UAT Cluster
├── Worker Node 1
├── Worker Node 2
└── Worker Node 3
```

When a Pod is created, the Kubernetes Scheduler selects a suitable worker node.

Creating a new Pod does **not** automatically mean creating a new worker node.

If no suitable node has enough capacity, the Pod can remain unscheduled until capacity becomes available.

---

## 13. Stateless vs Stateful

Spring Boot microservices are generally treated as stateless workloads:

```text
Deployment
   ↓
ReplicaSet
   ↓
Pods
```

PostgreSQL is stateful because it owns persistent data.

Conceptually:

```text
PostgreSQL
    ↓
Persistent Storage
```

---

## 14. StatefulSet

StatefulSet is designed for workloads requiring stable identity and state-related behavior.

Conceptually:

```text
StatefulSet
    ↓
PostgreSQL Pod
    ↓
Persistent Storage
```

StatefulSet is not simply a "better Deployment"; it addresses different workload characteristics.

---

## 15. Persistent Storage

Application Pods are generally disposable.

Database data must survive Pod replacement.

Therefore:

```text
PostgreSQL
    ↓
PersistentVolume
    ↓
Persistent Storage
```

The exact storage implementation depends on the Kubernetes environment.

---

## 16. ConfigMap

ConfigMap stores non-sensitive configuration.

KRB Enterprise examples:

```text
SPRING_PROFILES_ACTIVE=uat
DB_URL=jdbc:postgresql://postgres:5432/krb_enterprise
DB_USERNAME=krb
```

Conceptually:

```text
ConfigMap
├── SPRING_PROFILES_ACTIVE
├── DB_URL
└── DB_USERNAME
```

These values can be injected into containers.

---

## 17. Secret

Secret is intended for sensitive configuration.

Examples:

```text
DB_PASSWORD
API tokens
Private keys
Sensitive credentials
```

Conceptually:

```text
Secret
├── DB_PASSWORD
└── private-key.pem
```

Important:

> Kubernetes Secret still requires proper RBAC, access control, and encryption-at-rest/security configuration. It is not automatically equivalent to a dedicated enterprise secret vault.

---

## 18. Build Once, Configure Per Environment

Environment-specific configuration should not be baked into the application image.

```text
Same Docker Image
       │
       ├── DEV → ConfigMap + Secret
       ├── UAT → ConfigMap + Secret
       └── PROD → ConfigMap + Secret
```

Principle:

> Build once, configure per environment.

---

## 19. RSA Keys

KRB Enterprise uses RSA key files.

Sensitive key material should not be baked into the Docker image.

A Kubernetes Secret can provide the key material to the Pod as mounted files.

```text
Kubernetes Secret
       ↓
Pod
       ↓
/root/.krb-enterprise/secrets/
├── private-key.pem
└── public-key.pem
```

---

## 20. Health Probes

Kubernetes provides three important probes:

```text
Startup Probe
Liveness Probe
Readiness Probe
```

They answer different questions:

| Probe | Question |
|---|---|
| Startup | Has the application successfully started? |
| Liveness | Is the application still alive? |
| Readiness | Can the application receive traffic? |

---

## 21. Startup Probe

Startup Probe gives a slow-starting application time to initialize before normal liveness handling takes effect.

Without suitable startup handling:

```text
Application starting
   ↓
Liveness fails
   ↓
Container restarts
   ↓
Application starts again
```

---

## 22. Liveness Probe

Liveness determines whether Kubernetes should consider the application alive.

Repeated liveness failure can cause the container to be restarted.

Purpose:

> Recovery from an application that is no longer functioning correctly.

---

## 23. Readiness Probe

Readiness determines whether a Pod should receive traffic.

Example:

```text
Payment Service
├── Pod A → Ready
├── Pod B → NOT Ready
└── Pod C → Ready
```

Traffic goes only to ready Pods.

When Pod B becomes ready:

```text
Payment Service
├── Pod A → Ready
├── Pod B → Ready
└── Pod C → Ready
```

Pod B can now receive traffic.

---

## 24. Probes During Rolling Deployment

For a new version:

```text
New Pod
   ↓
Application starts
   ↓
Startup succeeds
   ↓
Readiness succeeds
   ↓
Pod becomes eligible for Service traffic
   ↓
Old Pod can be replaced
```

This is an important mechanism for safer rolling deployments.

---

## 25. Liveness vs Readiness

Remember:

```text
Liveness
"Should Kubernetes restart this container?"

Readiness
"Should this Pod receive traffic?"
```

A temporary database outage should not automatically cause all application Pods to restart.

Poorly designed liveness checks can create cascading failures.

---

## 26. Application-to-Database Communication

If PostgreSQL is exposed through a Kubernetes Service:

```text
core-krb-payment Pod
        ↓
postgres Service
        ↓
PostgreSQL
        ↓
Persistent Storage
```

The application uses the Service endpoint rather than the PostgreSQL Pod IP.

Current KRB Enterprise model:

```text
DB_URL=jdbc:postgresql://postgres:5432/krb_enterprise
```

---

## 27. Complete Mental Model

```text
                         UAT CLUSTER
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
    uat-core              uat-noncore             uat-ui
        │                     │                     │
   ┌────┼────┐            ┌───┴────┐           ┌───┴──────┐
   │    │    │            │        │           │          │
Payment Customer Order  Application Partner  Application Partner
                                               UI          UI
```

Application resources:

```text
Deployment
   ↓
ReplicaSet
   ↓
Pods
   ↓
Containers
```

Networking:

```text
Service
   ↓
Pods
```

Configuration:

```text
ConfigMap → non-sensitive configuration
Secret    → sensitive configuration
```

Database:

```text
Application
   ↓
PostgreSQL Service
   ↓
PostgreSQL
   ↓
Persistent Storage
```

Health:

```text
Startup → Liveness → Readiness → Service Traffic
```

---

## 28. Interview Takeaways

### Why Kubernetes if Docker exists?

Docker primarily solves containerization and container execution.

Kubernetes adds cluster-level orchestration:

- Scheduling
- Scaling
- Self-healing
- Service discovery
- Rolling deployments
- Desired-state management
- Workload management

Progression:

```text
Docker
  ↓
Docker Compose
  ↓
Kubernetes
```

### Deployment vs ReplicaSet

```text
Deployment
  ↓
Manages ReplicaSet
  ↓
ReplicaSet maintains Pods
```

### Pod vs Container

A Pod is the Kubernetes deployment unit and can contain one or more containers.

### Service vs Pod

Pod IPs can change.

A Service provides a stable endpoint and routes traffic to matching Pods.

### Deployment vs StatefulSet

```text
Deployment  → typically stateless applications
StatefulSet → stateful workloads requiring stable identity/storage behavior
```

### Liveness vs Readiness

```text
Liveness  → restart/recovery decision
Readiness → traffic decision
```

---

## 29. Next Session

Continue with:

1. CPU and Memory Requests
2. CPU and Memory Limits
3. How the Scheduler chooses a Worker Node
4. Resource management in the UAT cluster
5. RollingUpdate configuration
6. Rollback
7. Volumes / PersistentVolumes / PersistentVolumeClaims
8. StatefulSet in more depth
9. Jobs and CronJobs
10. Ingress
11. Helm
12. KRB Enterprise Kubernetes deployment
13. GitHub Actions → Kubernetes CD

---

## Session Status

```text
Cluster                         ✅
Control Plane / Worker Nodes   ✅
Namespace                      ✅
Pod                            ✅
Deployment                     ✅
ReplicaSet                     ✅
Service                        ✅
Kubernetes DNS                 ✅
ConfigMap                      ✅
Secret                         ✅
StatefulSet                    ✅
Persistent Storage             ✅
Startup Probe                  ✅
Liveness Probe                 ✅
Readiness Probe                ✅
Rolling Deployment             ✅
```

**Next topic:** CPU/Memory Requests & Limits → Kubernetes Scheduler
