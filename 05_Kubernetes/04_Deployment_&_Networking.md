# KRB Academy — Kubernetes Notes
## Session: 2026-09-12

## 1. Workload, Networking, and Storage Are Separate Concerns

- **Deployment / StatefulSet** manages workloads and Pods.
- **Service** provides stable networking to Pods.
- **PV / PVC** provides persistent storage.

These are separate Kubernetes concepts that combine to form a complete application architecture.

## 2. Deployment for KRB Enterprise

KRB Enterprise uses a Deployment because the application is currently treated as stateless.

```text
Deployment
    ↓
ReplicaSet
    ↓
Pods
```

With `replicas: 2`, Kubernetes maintains two application Pods.

The Pods are interchangeable from the application's perspective.

## 3. StatefulSet for PostgreSQL

PostgreSQL uses a StatefulSet because it is a stateful workload.

```text
StatefulSet
    ↓
postgres-0
```

The Pod has stable StatefulSet identity and persistent storage.

## 4. Service

A Service provides a stable network endpoint for Pods.

Pods have dynamically assigned IP addresses, so clients should not depend directly on Pod IPs.

KRB Enterprise:

```text
Service: krb-enterprise
Port: 8282
    ↓
Pod 1 / Pod 2
```

PostgreSQL:

```text
Service: postgres
Port: 5432
    ↓
postgres-0
```

Important distinction:

```text
KRB Enterprise = Deployment + Service
PostgreSQL     = StatefulSet + Service
```

A Service does not run an application; it provides networking to a workload.

## 5. ClusterIP

The KRB Enterprise Service uses:

```yaml
type: ClusterIP
```

ClusterIP makes the Service reachable from inside the Kubernetes cluster.

Current lab Service:

```text
10.103.207.143:8282
```

PostgreSQL Service:

```text
10.98.7.36:5432
```

The Service provides a stable virtual endpoint while Pod IPs can change.

## 6. Service Selectors and Endpoints

The KRB Enterprise Service uses:

```yaml
selector:
  app: krb-enterprise
```

This matches the labels on both KRB Enterprise Pods.

Verification showed:

```text
192.168.207.135:8282
192.168.207.136:8282
```

Therefore:

```text
KRB Service
    ├── Pod 1
    └── Pod 2
```

Kubernetes 1.33+ recommends EndpointSlice over the legacy Endpoints API.

## 7. PersistentVolume and PersistentVolumeClaim

PostgreSQL persistence follows:

```text
Pod
 ↓
PVC
 ↓
PV
 ↓
Node storage
```

A **PersistentVolume (PV)** represents storage available to Kubernetes.

A **PersistentVolumeClaim (PVC)** represents an application's request for storage.

The PostgreSQL StatefulSet creates its PVC through:

```yaml
volumeClaimTemplates:
  - metadata:
      name: postgres-data
```

The request is:

```text
5Gi
ReadWriteOnce
```

Kubernetes binds the PVC to a matching PV.

## 8. Why the PV and Container Paths Differ

The Kubernetes node path is:

```text
/var/lib/kubernetes/postgres
```

The PostgreSQL container mount path is:

```text
/var/lib/postgresql
```

They are intentionally different.

```text
Ubuntu Node
/var/lib/kubernetes/postgres
          │
          │ volume mount
          ▼
PostgreSQL Container
/var/lib/postgresql
```

The PV's `hostPath` is the node-side location. The `volumeMount` is the container-side location.

## 9. Why PostgreSQL Was Initially Pending

The PostgreSQL Pod initially remained:

```text
Pending
```

The scheduler reported:

```text
pod has unbound immediate PersistentVolumeClaims
```

The chain was:

```text
StatefulSet
    ↓
PVC requested
    ↓
No matching PV
    ↓
PVC unbound
    ↓
postgres-0 Pending
```

After creating the PV:

```text
PVC
 ↓
PV
 ↓
Bound
 ↓
postgres-0
 ↓
Running
```

## 10. PostgreSQL Networking

KRB Enterprise connects using:

```text
jdbc:postgresql://postgres:5432/krb_enterprise
```

`postgres` is the Kubernetes Service name.

```text
KRB Enterprise Pod
        ↓
postgres:5432
        ↓
PostgreSQL Service
        ↓
postgres-0
```

The application does not need to know the PostgreSQL Pod IP.

## 11. Accessing a ClusterIP Service from Windows

ClusterIP is internal to Kubernetes.

For the single-node lab, access was provided using:

```bash
kubectl port-forward --address 0.0.0.0 -n uat-core svc/krb-enterprise 8282:8282
```

Default port-forwarding listens on `127.0.0.1`. Using `--address 0.0.0.0` allows traffic arriving through the VM network interface.

The complete lab path is:

```text
Windows / Postman
        ↓
VirtualBox NAT
        ↓
Ubuntu VM :8282
        ↓
kubectl port-forward
        ↓
KRB Enterprise Service
        ↓
KRB Enterprise Pods
```

This is a lab access mechanism, not the intended production exposure pattern.

## 12. Current Kubernetes Architecture

```text
Kubernetes Cluster
│
└── uat-core
    │
    ├── KRB Enterprise
    │   ├── Deployment
    │   │   └── 2 Pods
    │   └── Service
    │       └── ClusterIP :8282
    │
    └── PostgreSQL
        ├── StatefulSet
        │   └── postgres-0
        ├── Service
        │   └── ClusterIP :5432
        └── Storage
            └── PVC → PV → hostPath
```

## 13. Key Mental Models

### Workload

```text
Deployment → ReplicaSet → Pods
StatefulSet → Stateful Pods
```

### Networking

```text
Service → Pods
```

### Storage

```text
Pod → PVC → PV → Storage
```

### Complete application

```text
Client
  ↓
Service
  ↓
Deployment
  ↓
Pods
  ↓
Database Service
  ↓
StatefulSet
  ↓
Persistent Storage
```
