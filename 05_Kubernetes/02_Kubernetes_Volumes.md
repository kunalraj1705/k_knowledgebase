# KRB Academy — Kubernetes Notes
## Session: 2026-09-10

## 1. Kubernetes Storage — Volumes, PV and PVC

Pods and containers are disposable. Data written only to a container filesystem may disappear when the container is removed and recreated.

Stateless applications such as Spring Boot generally do not need persistent application storage. Stateful workloads such as PostgreSQL must keep database data outside the disposable container filesystem.

Core model:

```text
Application Pod
      |
      v
     PVC
      |
      v
     PV
      |
      v
Storage Backend
```

### PVC — PersistentVolumeClaim
The application's request for persistent storage.

Example:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

### PV — PersistentVolume
A Kubernetes storage resource representing storage made available to workloads.

### StorageClass
Defines/provisions storage dynamically.

Conceptually:

```text
PVC
 |
 v
StorageClass
 |
 v
Dynamic provisioning
 |
 v
PV
 |
 v
Actual storage
```

---

## 2. Docker Persistent Volume — KRB Enterprise Example

The KRB Enterprise Docker Compose configuration uses a Docker named volume:

```yaml
services:
  postgres:
    image: postgres:18
    volumes:
      - postgres_data:/var/lib/postgresql

volumes:
  postgres_data:
```

Syntax:

```text
SOURCE : DESTINATION
```

Therefore:

```text
postgres_data
      |
      v
/var/lib/postgresql
```

`postgres_data` is the Docker-managed named volume.

`/var/lib/postgresql` is the path inside the PostgreSQL container where the volume is mounted.

PostgreSQL writes database files to that path, while Docker stores the data in the named volume.

### Persistence behavior

Removing the container:

```bash
docker rm postgres
```

does not normally remove the named volume.

However:

```bash
docker-compose down -v
```

also removes the volumes, so persistent database data can be deleted.

### Compose project naming

The logical volume name in Compose:

```yaml
volumes:
  postgres_data:
```

may become an actual Docker volume name prefixed by the Compose project name.

For example:

```text
deploy + postgres_data
        |
        v
deploy_postgres_data
```

The currently running KRB Enterprise PostgreSQL container was verified with:

```bash
docker inspect postgres --format '{{range .Mounts}}{{println .Name "->" .Destination}}{{end}}'
```

and showed:

```text
deploy_postgres_data -> /var/lib/postgresql
```

Therefore `deploy_postgres_data` is the volume currently attached to the running PostgreSQL container.

---

## 3. Docker Mount Types

### Named Volume

```yaml
volumes:
  postgres_data:

services:
  postgres:
    volumes:
      - postgres_data:/var/lib/postgresql
```

Docker manages the storage location. It is commonly used for databases and persistent application data.

### Bind Mount

A host filesystem path is mounted into the container:

```yaml
volumes:
  - ./config:/app/config
```

KRB Enterprise RSA key mounts are bind mounts:

```yaml
volumes:
  - /home/ubuntu/workspace/k_enterprise/secrets/private-key.pem:/root/.krb-enterprise/secrets/private-key.pem:ro
  - /home/ubuntu/workspace/k_enterprise/secrets/public-key.pem:/root/.krb-enterprise/secrets/public-key.pem:ro
```

`:ro` means read-only.

### tmpfs Mount

Stores data in memory:

```yaml
tmpfs:
  - /tmp
```

The data does not survive container termination.

### Comparison

| Type | Storage | Survives container deletion? | Common use |
|---|---|---:|---|
| Named volume | Docker-managed | Yes | PostgreSQL, Redis |
| Bind mount | Host filesystem | Yes | Config, source, certificates |
| tmpfs | RAM | No | Temporary data |

---

## 4. Kubernetes Access Modes

### ReadWriteOnce — RWO

Storage can be mounted read/write by workloads on one node.

Important: RWO refers to one node, not necessarily one Pod.

### ReadOnlyMany — ROX

Multiple nodes can mount the storage read-only.

### ReadWriteMany — RWX

Multiple nodes can mount the storage read/write.

The underlying storage backend must support the requested access mode.

---

## 5. PostgreSQL: Deployment vs StatefulSet

Spring Boot applications are generally stateless:

```text
Deployment
    |
    v
Pods
```

PostgreSQL is stateful:

```text
StatefulSet
    |
    v
PostgreSQL Pod
    |
    v
PVC
    |
    v
Persistent Storage
```

StatefulSet is suitable for workloads requiring stable identity and persistent storage behavior.

If the PostgreSQL Pod is recreated, the persistent storage can be attached again so database data survives Pod recreation.

---

## 6. PVC vs PV vs StorageClass

Remember:

```text
PVC = "I need storage"
PV  = "Here is storage"
StorageClass = "This is how storage is provisioned"
```

Application workloads generally request storage through PVC instead of directly managing the underlying disk.

---

## 7. Pod Deletion vs PVC Deletion

Deleting a Pod:

```text
Pod deleted
    |
    v
New Pod created
    |
    v
PVC remains
    |
    v
Persistent data remains
```

Deleting the PVC is different:

```bash
kubectl delete pvc postgres-data
```

The effect on the underlying storage depends on the PV/storage reclaim policy.

Common policies:

- `Retain` — underlying storage is retained.
- `Delete` — provisioned underlying storage may be deleted when the claim is deleted, depending on the storage provisioner.

---

## 8. KRB Enterprise Kubernetes Storage Architecture

Current KRB Enterprise uses one PostgreSQL database named `krb_enterprise`.

Conceptual Kubernetes architecture:

```text
UAT Kubernetes Cluster
|
+-- uat-core
|    +-- KRB application workloads
|
+-- uat-noncore
|    +-- Non-core workloads
|
+-- uat-ui
|    +-- UI workloads
|
+-- PostgreSQL
     +-- StatefulSet
     +-- PostgreSQL Pod
     +-- PostgreSQL Service
     +-- PVC
          |
          +-- Persistent Storage
```

The application should connect to the PostgreSQL Kubernetes Service rather than directly to a PostgreSQL Pod IP.

---

# 9. Kubernetes ConfigMap

A ConfigMap stores non-sensitive configuration.

KRB Enterprise examples:

```text
SPRING_PROFILES_ACTIVE
DB_URL
DB_USERNAME
```

Example:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: krb-enterprise-config
data:
  SPRING_PROFILES_ACTIVE: "uat"
  DB_URL: "jdbc:postgresql://postgres:5432/krb_enterprise"
  DB_USERNAME: "krb"
```

---

# 10. Kubernetes Secret

A Secret is intended for sensitive configuration.

KRB Enterprise examples:

```text
DB_PASSWORD
RSA private key
credentials
tokens
```

Example:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: krb-enterprise-secret
type: Opaque
stringData:
  DB_PASSWORD: "password"
```

Important: Kubernetes Secrets are not automatically equivalent to a dedicated enterprise secrets vault. RBAC, encryption at rest, restricted access, and appropriate secret-management practices are still important.

---

# 11. Injecting ConfigMap and Secret into a Pod

A Deployment can import values as environment variables:

```yaml
containers:
  - name: krb-enterprise
    image: ghcr.io/kunalraj1705/krbenterprise:IMAGE_TAG

    envFrom:
      - configMapRef:
          name: krb-enterprise-config
      - secretRef:
          name: krb-enterprise-secret
```

The container receives values such as:

```text
SPRING_PROFILES_ACTIVE=uat
DB_URL=jdbc:postgresql://postgres:5432/krb_enterprise
DB_USERNAME=krb
DB_PASSWORD=password
```

Spring Boot can read these environment variables.

---

# 12. Build Once, Configure Per Environment

The same application image can be promoted across environments:

```text
                 SAME IMAGE
                     |
       +-------------+-------------+
       |             |             |
      DEV           UAT           PROD
       |             |             |
   ConfigMap      ConfigMap     ConfigMap
   Secret         Secret        Secret
```

Principle:

```text
Application artifact = immutable
Configuration        = environment-specific
```

---

# 13. RSA Keys in Kubernetes

The current Docker Compose implementation uses bind mounts:

```text
Host private-key.pem
        |
        v
Container
/root/.krb-enterprise/secrets/private-key.pem
```

In Kubernetes, the intended model is to store sensitive key material in a Kubernetes Secret and mount it into the Pod as files:

```text
Kubernetes Secret
       |
       | mounted as files
       v
Pod
 |
 +-- /root/.krb-enterprise/secrets/
       +-- private-key.pem
       +-- public-key.pem
```

This avoids depending on a specific worker-node filesystem path.

---

# 14. Core Mental Models

### Docker persistence

```text
Named Volume
     |
     v
Container filesystem path
```

### Kubernetes persistence

```text
PVC
 |
 v
PV
 |
 v
Storage Backend
```

### Kubernetes configuration

```text
ConfigMap
   |
   v
Non-sensitive configuration

Secret
   |
   v
Sensitive configuration
```

### Application architecture

```text
Deployment
    |
    v
Stateless application Pods

StatefulSet
    |
    v
Stateful workload
    |
    v
PVC
    |
    v
Persistent storage
```

---

# 15. Kubernetes Track Progress

```text
Cluster                         ✅
Namespace                       ✅
Deployment                      ✅
ReplicaSet                      ✅
Pod                             ✅
Service                         ✅
DNS                             ✅
StatefulSet                     ✅
Persistent storage concepts     ✅
ConfigMap                       ✅
Secret                          ✅
Health probes                   ✅
Resource requests/limits        ✅
Scheduler                       ✅
Rolling updates                 ✅
Rollback                        ✅
Volumes / PV / PVC              ✅
```

## Next Topics

```text
Service types
    |
    +-- ClusterIP
    +-- NodePort
    +-- LoadBalancer
    |
    v
Kubernetes networking in depth
    |
    v
Ingress
    |
    v
HPA / Autoscaling
    |
    v
RBAC
    |
    v
Kubernetes security
    |
    v
KRB Enterprise Kubernetes deployment
```
