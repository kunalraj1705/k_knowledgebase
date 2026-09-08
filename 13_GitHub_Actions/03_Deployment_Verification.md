# KRB Academy Notes — 2026-09-08

## GitHub Actions Completion and Kubernetes Introduction

### 1. Deployment Verification
Container status alone does not prove that an application deployment succeeded.

A stronger flow is:
Build → Test → Image → Publish → Deploy → Application Health Check

KRB Enterprise uses Spring Boot Actuator for application-level verification.

### 2. Health Checks
The configured health endpoint is:
`/krbenterprise/actuator/health`

A successful deployment returns HTTP 200 and `status: UP`, with liveness and readiness groups available.

CI/CD can repeatedly check the endpoint during startup and fail the deployment when the application does not become healthy.

### 3. GitHub Actions Variables and Secrets
Deployment configuration separates:
- Non-sensitive configuration → environment variables
- Sensitive values → secrets

UAT uses:
- `SPRING_PROFILES_ACTIVE`
- `DB_URL`
- `DB_USERNAME`
- `DB_PASSWORD` as a secret

### 4. Immutable Image Versioning
Git commit SHA is used as an immutable image tag.

Flow:
`Git commit → Docker image → SHA tag → UAT`

This identifies exactly which source revision is deployed and improves reproducibility.

### 5. Safe Deployment Principle
A deployment should not be declared successful merely because a container starts.

The candidate version should pass an application-level health check before being considered successful.

### 6. Rolling Deployment
A rolling deployment keeps the old version available while introducing the new version:

`Old Pods → Start New Pod(s) → New Pod Ready → Traffic to New → Remove Old`

The old version should not be removed before the replacement is validated.

### 7. Why Kubernetes
Docker Compose on a single host cannot have two containers simultaneously publish the same host port.

Kubernetes solves this using Pod networking and Services:
- Pods have individual IP addresses.
- Multiple Pods can listen on the same container port.
- A Service provides one stable endpoint.
- Service selection routes traffic to eligible Pods.

Conceptually:
`Client → Service :8282 → Pod A :8282 / Pod B :8282`

### 8. Kubernetes Desired State
The foundational Kubernetes model introduced today is desired state plus reconciliation.

For example:
`replicas: 3`

declares the desired number of application instances. Kubernetes controllers continuously reconcile actual state toward that desired state.

### 9. Kubernetes Topics Ahead
- Cluster and Control Plane
- Worker Nodes
- kubectl
- Namespaces
- Pods
- Deployments
- ReplicaSets
- Services and service discovery
- ConfigMaps
- Secrets
- Resource requests/limits
- Liveness/readiness/startup probes
- Rolling updates
- Rollbacks
- Volumes/Persistent Volumes
- StatefulSets
- Jobs/CronJobs
- Ingress
- Helm
- KRB Enterprise Kubernetes deployment
- GitHub Actions → Kubernetes CD

## Key Principles
1. CI verifies before deployment.
2. Git SHA tags make deployments reproducible.
3. Application health is stronger verification than container status.
4. Secrets hold sensitive deployment values.
5. Rolling deployment keeps the previous version available while validating the new one.
6. Kubernetes Service provides stable networking while Pods are replaced.
7. Readiness determines whether a Pod can receive traffic.
8. Kubernetes desired state and reconciliation underpin self-healing and controlled deployment.

## Track Status
**GitHub Actions: Complete for current KRB Enterprise scope.**

**Next track: Kubernetes.**
