# KRB Academy — AWS Notes
## S3 Security, CloudTrail, CloudWatch & EKS Foundations

### 1. S3 Storage Classes
Choose a storage class based on access frequency, retrieval needs, cost, and retention.

- **S3 Standard** → frequently accessed data.
- **S3 Standard-IA** → infrequently accessed data.
- **S3 Glacier** → long-term archival / rarely accessed data.

Mental model:

```text
Frequently accessed → Standard
Less frequently accessed → Standard-IA
Long-term archive → Glacier
```

### 2. S3 Versioning
Versioning keeps multiple versions of an object.

```text
krb-document.pdf
 ├── Version 1
 ├── Version 2
 └── Version 3 ← current
```

Useful for accidental overwrites, accidental deletions, recovery, and maintaining object history.

**Important:** Versioning is not a complete audit log of who performed every action. CloudTrail is used for AWS activity auditing.

### 3. S3 Encryption
Server-side encryption protects S3 data at rest.

**SSE-S3**
```text
S3 → AWS-managed encryption
```

**SSE-KMS**
```text
S3 → AWS KMS → KMS key
```

SSE-KMS provides additional key-management control and auditing capabilities. When KMS encryption is involved, appropriate KMS permissions may also be required.

### 4. IAM Policy vs S3 Bucket Policy

**IAM / identity policy**
> What permissions does this identity have?

```text
EC2 → IAM Role → Identity Policy → s3:GetObject
```

**S3 bucket policy**
> Who/what can access this resource, and what can they do?

```text
S3 Bucket → Bucket Policy → Principal → Action → Resource
```

### 5. Principal
`Principal` identifies the entity a resource-based policy applies to.

Example:

```json
"Principal": {
  "AWS": "arn:aws:iam::<ACCOUNT_ID>:role/krb-academy-ec2-role"
}
```

### 6. Explicit Deny
A critical IAM rule:

> **Explicit Deny overrides Allow.**

```text
Allow + Explicit Deny → DENIED
```

### 7. S3 Lifecycle Rules
Lifecycle rules automate object transitions and expiration.

Example:

```text
Standard
   ↓
Standard-IA
   ↓
Glacier
   ↓
Delete after required retention period
```

Benefits:
- Avoid manual management of huge numbers of objects.
- Reduce storage cost for older data.
- Automate retention.
- Automatically expire data when permitted.

Before creating a lifecycle rule, determine:
```text
Required retention
↓
Access frequency
↓
Transition point
↓
Deletion point
```

### 8. S3 Block Public Access
KRB Enterprise application data, logs, and user documents should normally remain private.

Preferred:

```text
KRB Enterprise → IAM Role → Private S3 Bucket → Objects
```

Avoid making a bucket public simply because an application needs access. S3 Block Public Access helps prevent unintended public exposure.

### 9. S3 Presigned URLs
A presigned URL provides temporary, limited access to an S3 operation without giving the client AWS credentials or making the bucket public.

```text
User
 ↓
KRB Enterprise
 ↓
Authentication
 ↓
Authorization
 ↓
Generate presigned URL
 ↓
S3 object
```

Example:
```text
URL valid for 10 minutes
```

The application should still authorize the user before generating the URL.

### 10. AWS CloudTrail
CloudTrail records AWS API activity and supports auditing.

It helps answer:
- Who performed an action?
- What action?
- Which resource?
- When?
- Other event details such as source and outcome.

Mental model:

```text
IAM       → Can this identity perform the action?
CloudTrail → What AWS API activity actually happened?
```

Example:
> To investigate who deleted `krb-test.txt`, investigate CloudTrail.

### 11. AWS CloudWatch
CloudWatch is used for monitoring and observability.

Capabilities learned:
- Metrics
- Logs
- Alarms

```text
AWS/Application
      ↓
 CloudWatch
 ┌────┼────┐
Metrics Logs Alarms
```

**Metrics** answer:
> What is happening?

**Alarms** answer:
> When should we alert?

**Logs** answer:
> Why did it happen?

### 12. Production Metrics
For KRB Enterprise, important monitoring signals include:

```text
Request rate → How much traffic?
Latency      → How fast?
Error rate   → How reliable?
CPU/Memory   → How is infrastructure behaving?
Database     → Is the DB a bottleneck?
```

CPU alone is not enough; an application can have acceptable CPU utilization while users experience high latency.

### 13. Average Latency vs Percentiles

Example:

```text
100 ms
100 ms
100 ms
100 ms
1000 ms
```

Average = 280 ms.

The average can hide slow requests.

**p50**
- 50% of requests are at or below this latency.
- Median.

**p95**
- 95% are at or below this latency.
- Slowest 5% are above it.

**p99**
- 99% are at or below this latency.
- Slowest 1% are above it.

Example:

```text
Average = 150 ms
p99 = 2500 ms
```

2,500 / 150 ≈ **16.7×** the average.

With 100,000 requests, approximately 1% = 1,000 requests could experience around 2.5 seconds.

Key lesson:
> Average latency alone can hide serious user-experience problems.

### 14. KRB Enterprise AWS Direction
KRB Enterprise is already deployed through GitHub CI/CD and Kubernetes on an Ubuntu VM. We do **not** need to redo the application deployment.

Current:

```text
GitHub
 ↓
GitHub CI/CD
 ↓
Kubernetes
 ↓
Ubuntu VM
 ↓
KRB Enterprise
```

AWS direction:

```text
GitHub
 ↓
GitHub Actions
 ↓
Docker Image
 ↓
Amazon ECR
 ↓
EKS
 ↓
KRB Enterprise
```

### 15. EKS
EKS = Amazon Elastic Kubernetes Service.

EKS is AWS's managed Kubernetes service.

Existing:

```text
Ubuntu VM
 ↓
Kubernetes
 ↓
KRB Enterprise
```

AWS:

```text
AWS
 ↓
EKS
 ↓
Kubernetes
 ↓
KRB Enterprise
```

Kubernetes concepts remain:
- Pods
- Deployments
- Services
- ConfigMaps
- Secrets
- Namespaces
- Ingress

AWS manages the EKS control plane. Worker compute is still required for Pods.

A common model:

```text
EKS
 ├── AWS-managed Control Plane
 └── EC2 Worker Nodes
       ├── Pod
       ├── Pod
       └── Pod
```

### 16. EKS + IAM
Kubernetes workloads on AWS need secure AWS identity when accessing AWS services.

Avoid embedding long-lived credentials:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

inside Kubernetes workloads.

EKS provides mechanisms such as **EKS Pod Identity** to associate AWS permissions with workloads.

### 17. Next Topic — ECR
ECR = Amazon Elastic Container Registry.

It is AWS's managed container image registry.

Target flow:

```text
GitHub Actions
 ↓
Docker build
 ↓
Amazon ECR
 ↓
EKS
 ↓
KRB Enterprise Pod
```

### Next Session
Resume from:

```text
Amazon ECR
 ↓
Container image repository
 ↓
GitHub Actions → ECR
 ↓
EKS pulls image
 ↓
KRB Enterprise Pod
```

Then continue into:
```text
EKS Cluster
 ↓
Node Groups / Compute
 ↓
Kubernetes Deployment
 ↓
Service
 ↓
Ingress / Load Balancer
 ↓
RDS
 ↓
CloudWatch
```
