# KRB Academy — Kubernetes Notes
## Session: 2026-09-11

## Topics Covered

### HPA — Horizontal Pod Autoscaler
HPA automatically adjusts Pod replica count based on workload metrics.

```text
Traffic → Service → Pods
              ↑
              HPA observes metrics
              ↓
       Deployment replicas
```

HPA scales Pods, not worker nodes.

```text
HPA → Pods
Cluster Autoscaler → Nodes
```

CPU-based HPA commonly evaluates utilization relative to CPU requests. Example: a 500m CPU request with 400m usage represents 80% utilization.

Example:
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: krb-enterprise
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: krb-enterprise
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
```

Useful commands:
```bash
kubectl get hpa -n uat-core
kubectl describe hpa krb-enterprise -n uat-core
kubectl top pods -n uat-core
kubectl get pods -n uat-core -w
```

`kubectl top` requires a metrics provider such as Metrics Server.

### Kubernetes Security
Kubernetes security and application security are separate layers.

```text
Kubernetes RBAC → Kubernetes resources
Application RBAC → Business/API authorization
```

Authentication asks **who are you?** Authorization asks **what are you allowed to do?**

RBAC resources covered:
- Role
- RoleBinding
- ClusterRole
- ClusterRoleBinding
- ServiceAccount

A Role defines permissions within a namespace. A RoleBinding assigns those permissions to a subject.

Example:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: uat-core
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

ServiceAccounts provide Kubernetes identities for workloads. Least privilege is preferred over broad permissions such as `cluster-admin`.

### Kubernetes Secrets
Secrets can hold sensitive runtime configuration such as DB passwords and RSA private keys.

For KRB Enterprise:
```text
Kubernetes Secret
      ↓
     Pod
      ↓
Application
```

RSA private keys should not be baked into the Docker image.

A Kubernetes Secret is not automatically equivalent to a dedicated enterprise vault; RBAC, encryption at rest, access control, auditing, and rotation remain important.

### Practical Kubernetes Direction
We decided to move from Kubernetes theory to actual KRB Enterprise deployment.

Target:
```text
Docker Image
   ↓
Deployment
   ↓
Pods
   ↓
Service
   ↓
ConfigMap / Secret
   ↓
PostgreSQL
   ↓
Ingress
   ↓
HPA
```

### Kubernetes Host Preparation
The Ubuntu VM was assessed for a single-node learning cluster:
- Ubuntu 26.04.1 LTS
- x86_64
- 8 CPUs
- 7.2 GiB RAM
- ~29 GiB free disk
- No swap

Existing runtime:
- Docker 29.1.3
- containerd 2.2.2, active

Kubernetes will use containerd as the CRI-compatible runtime rather than Docker Engine directly.

Installed:
- kubeadm v1.37.0
- kubelet v1.37.0
- kubectl v1.37.0

The Kubernetes v1.37 APT repository was configured successfully.

A default containerd configuration was generated at:
```text
/etc/containerd/config.toml
```

The next configuration step is enabling `SystemdCgroup = true`, followed by restarting containerd, enabling kubelet, and initializing the cluster.
