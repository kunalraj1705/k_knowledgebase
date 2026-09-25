# KRB Academy --- AWS Complete Notes

## AWS Foundations → Networking → EC2 → IAM → S3

### Consolidated notes from the completed sessions

------------------------------------------------------------------------

# 1. AWS Global Infrastructure

## Region

A Region is a geographical boundary in AWS where AWS resources and
operations are deployed.

``` text
Region
  ↓
Geographical boundary
  ↓
Contains multiple Availability Zones
```

## Availability Zone (AZ)

An Availability Zone is a distinct location/infrastructure area inside
an AWS Region.

A Region is divided into multiple Availability Zones.

Using multiple AZs improves high availability:

``` text
                 Region
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
        AZ-1                 AZ-2
          │                   │
       Instance            Instance
          │                   │
          └─────────┬─────────┘
                    │
              Application
```

If one AZ becomes unavailable, another AZ can continue serving requests.

------------------------------------------------------------------------

# 2. VPC and Subnets

## VPC

VPC = Virtual Private Cloud.

A VPC provides an isolated virtual networking environment for AWS
resources.

``` text
AWS Region
   ↓
VPC
   ├── Public Subnet
   └── Private Subnet
```

Different customers can have separate VPCs even when their resources are
running in the same AWS infrastructure.

## Subnet

A subnet is a subdivision of a VPC.

Typical application architecture:

``` text
Internet
   ↓
Load Balancer
   ↓
Private Backend
   ↓
PostgreSQL
```

Public subnets contain resources intended to have a route to/from the
internet. Private subnets are used for resources that should not be
directly reachable from the internet.

------------------------------------------------------------------------

# 3. Route Tables

A Route Table directs network traffic toward the appropriate
destination.

``` text
Request
   ↓
Route Table
   ↓
Appropriate destination
```

------------------------------------------------------------------------

# 4. Internet Gateway

An Internet Gateway (IGW) provides the connection between a VPC and the
internet.

``` text
Internet
   ↕
Internet Gateway
   ↕
VPC
```

------------------------------------------------------------------------

# 5. NAT Gateway

A NAT Gateway supports outbound internet connectivity from private
resources without making those private resources directly reachable from
the internet.

``` text
Private Backend
      │
      ▼
 NAT Gateway
      │
      ▼
  Internet
```

Desired model:

``` text
Internet → Backend
     ❌

Backend → Internet
     ✅
```

------------------------------------------------------------------------

# 6. Load Balancer and Security Groups

The public-facing entry point is the Load Balancer.

``` text
Internet
   ↓
Load Balancer
   ↓
Backend EC2
   ↓
PostgreSQL
```

Example port model used in learning:

``` text
Load Balancer → Backend : 8080
Backend → PostgreSQL     : 5432
```

Security rules should restrict traffic by source and port.

``` text
Internet → Backend:8080
    ❌

Load Balancer → Backend:8080
    ✅

Backend → PostgreSQL:5432
    ✅
```

Principle:

> Only allow communication paths that are actually required.

------------------------------------------------------------------------

# 7. EC2

EC2 = Elastic Compute Cloud.

EC2 provides virtual compute resources such as:

-   vCPU
-   Memory
-   Network
-   Attached storage such as EBS

``` text
EC2
├── vCPU
├── RAM
├── Network
└── EBS
```

------------------------------------------------------------------------

# 8. AMI

AMI = Amazon Machine Image.

An AMI is a blueprint used to launch EC2 instances. It can contain the
operating system and required software/configuration.

``` text
AMI
 ↓
Blueprint
 ↓
EC2 Instance
```

It is useful when multiple instances need the same base environment.

------------------------------------------------------------------------

# 9. EC2 Instance Types

## T series

Designed for workloads with occasional CPU bursts.

``` text
Normal usage
   ↓
CPU credits accumulate
   ↓
CPU spike
   ↓
Credits can be consumed
```

## M series

General-purpose instances with a balanced CPU/memory profile.

## C series

Compute-optimized instances for CPU-intensive workloads.

## R series

Memory-optimized instances for memory-intensive workloads.

## Horizontal scaling

Instead of continually increasing the size of one instance:

``` text
Load
 ↓
Threshold exceeded
 ↓
Add another instance
```

------------------------------------------------------------------------

# 10. CPU Credits

For burstable T-series instances:

``` text
Usage below baseline
      ↓
Credits accumulate

Usage above baseline
      ↓
Credits consumed
```

CPU credits provide temporary burst capacity.

------------------------------------------------------------------------

# 11. Spot Instances

Spot Instances can be useful for workloads that are flexible and can
tolerate interruption.

Example:

``` text
Batch workload
     ↓
Doesn't run continuously
     ↓
Spot can be considered
```

AWS can reclaim Spot capacity, so workloads must tolerate interruption.

------------------------------------------------------------------------

# 12. EBS

EBS = Elastic Block Store.

EBS provides block storage for EC2 instances.

``` text
EC2
 │
 └── EBS
       └── OS / application data
```

EBS data persists when an EC2 instance is stopped.

We verified this practically after Stop → Start.

An observed root volume was:

``` text
/dev/nvme0n1p1
8.0G
1.7G used
6.4G available
```

EBS volumes are tied to an Availability Zone and cannot simply be
attached directly to an instance in another AZ.

------------------------------------------------------------------------

# 13. EC2 Stop vs Terminate

## Stop

``` text
Running
   ↓
Stop
   ↓
Stopped
```

Compute is paused and EBS remains.

After starting again, the instance can continue using the EBS data.

The public IPv4 address can change after Stop/Start.

## Terminate

``` text
Running
   ↓
Terminate
   ↓
Terminated
```

The EC2 instance is deleted. Associated resources such as EBS depend on
their deletion configuration.

------------------------------------------------------------------------

# 14. Public IP vs Private IP

## Private IP

Used for communication inside the VPC/private network.

## Public IP

Can be used for internet-based connectivity when networking and security
rules permit it.

For KRB Enterprise, the backend should not be directly exposed:

``` text
Internet
   ↓
Load Balancer
   ↓
Private Backend
```

------------------------------------------------------------------------

# 15. ENI

ENI = Elastic Network Interface.

It is the virtual network interface associated with an EC2 instance.

Physical analogy:

``` text
Physical machine → Network card
EC2              → ENI
```

------------------------------------------------------------------------

# 16. DNS

DNS = Domain Name System.

DNS resolves hostnames/domain names to network destinations.

``` text
Domain name
   ↓
DNS
   ↓
Endpoint / IP
```

Application flow:

``` text
User
 ↓
Domain
 ↓
Load Balancer
 ↓
Backend
```

------------------------------------------------------------------------

# 17. AWS Systems Manager

AWS Systems Manager can be used to manage EC2 instances without
requiring direct public SSH exposure in appropriate configurations.

This supports the goal of keeping EC2 private.

------------------------------------------------------------------------

# 18. EC2 Practical Lab

We launched an EC2 instance for KRB Academy.

We connected using SSH:

``` text
Windows
   ↓
SSH
   ↓
EC2
```

We also connected successfully from the iPad:

``` text
iPad
   ↓
SSH
   ↓
EC2
```

Commands used:

``` bash
whoami
hostname
uname -a
df -h
free -h
```

The connected user was:

``` text
ec2-user
```

We also encountered and fixed a Windows SSH private-key permission
problem:

``` text
Bad permissions
UNPROTECTED PRIVATE KEY FILE
```

The issue was that the key was accessible to additional Windows
identities.

------------------------------------------------------------------------

# 19. Security Groups

Security Groups provide network access control for resources such as
EC2.

SSH was configured through an inbound rule:

``` text
SSH / TCP 22
Source: My IP
```

If the client public IP changes, SSH can time out until the Security
Group source is updated.

We experienced this during the lab.

------------------------------------------------------------------------

# 20. IAM

IAM = Identity and Access Management.

Two foundational concepts:

``` text
Authentication
    ↓
"Who are you?"

Authorization
    ↓
"Are you allowed to do this?"
```

------------------------------------------------------------------------

# 21. IAM Users

An IAM User represents a person or identity that can have credentials
and permissions.

``` text
Kunal
 ↓
IAM User
 ↓
Permissions
```

------------------------------------------------------------------------

# 22. IAM Groups

A Group is a collection of IAM users.

``` text
Developer 1 ─┐
Developer 2 ─┼── Developers Group
Developer 3 ─┘
```

Groups make common permission management easier.

------------------------------------------------------------------------

# 23. IAM Roles

A Role is an identity that can be assumed.

For EC2 workloads:

``` text
EC2
 ↓
IAM Role
 ↓
AWS permissions
```

For KRB Enterprise:

``` text
KRB Enterprise
     ↓
EC2
     ↓
IAM Role
     ↓
Temporary credentials
     ↓
AWS services
```

This avoids embedding long-lived AWS credentials in the application when
a role can be used.

------------------------------------------------------------------------

# 24. IAM Policies

Policies define permissions.

Four important elements:

``` text
Effect
Action
Resource
Condition
```

Example:

``` json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::krb-enterprise-logs-app/*"
    }
  ]
}
```

### Effect

``` text
Allow
Deny
```

### Action

Defines the operation:

``` text
s3:GetObject
s3:PutObject
s3:DeleteObject
```

### Resource

Defines which resource the permission applies to.

### Condition

Adds additional restrictions under which the permission applies.

------------------------------------------------------------------------

# 25. Least Privilege

Principle:

> Give an identity only the permissions it actually needs.

Example:

``` text
s3:GetObject
    ✅

s3:PutObject
    ❌ unless required

s3:DeleteObject
    ❌ unless required
```

------------------------------------------------------------------------

# 26. Explicit Deny

Important IAM rule:

> An explicit Deny overrides an Allow.

``` text
Allow + Explicit Deny
        ↓
      DENIED
```

------------------------------------------------------------------------

# 27. Trust Policy vs Permissions Policy

This was a key IAM concept.

## Trust Policy

Answers:

> Who can assume this role?

## Permissions Policy

Answers:

> What can the role do?

``` text
IAM Role
├── Trust Policy
│     ↓
│   WHO can assume?
│
└── Permissions Policy
      ↓
    WHAT can they do?
```

------------------------------------------------------------------------

# 28. EC2 Trust Policy

Our EC2 role used a trust policy equivalent to:

``` json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sts:AssumeRole"
      ],
      "Principal": {
        "Service": [
          "ec2.amazonaws.com"
        ]
      }
    }
  ]
}
```

Interpretation:

``` text
Principal
    ↓
ec2.amazonaws.com
    ↓
EC2 is trusted

Action
    ↓
sts:AssumeRole
    ↓
EC2 can assume the role
```

This trust policy does not itself grant S3 permissions.

------------------------------------------------------------------------

# 29. IAM Hands-on --- EC2 to S3

We created an IAM role:

``` text
krb-academy-ec2-role
```

We used a limited policy:

``` text
krb-academy-s3-read
```

The policy allowed:

``` text
s3:GetObject
```

on:

``` text
arn:aws:s3:::krb-enterprise-logs-app/*
```

The role was attached to EC2.

Inside EC2 we ran:

``` bash
aws sts get-caller-identity
```

The returned ARN contained:

``` text
assumed-role/krb-academy-ec2-role/i-...
```

This proved that EC2 was using the IAM role.

No AWS access key or secret key was configured on the instance.

------------------------------------------------------------------------

# 30. IAM → S3 Permission Test

We successfully read an S3 object using:

``` text
s3:GetObject
```

We then attempted:

``` text
s3:DeleteObject
```

and received AccessDenied.

Therefore:

``` text
s3:GetObject
     ↓
     ✅

s3:DeleteObject
     ↓
     ❌
```

This experimentally proved least privilege.

Complete flow:

``` text
EC2
 ↓
IAM Role
 ↓
Temporary credentials
 ↓
IAM Policy
 ↓
s3:GetObject
 ↓
S3 object
 ↓
READ ✅
```

------------------------------------------------------------------------

# 31. S3 Fundamentals

S3 = Simple Storage Service.

S3 is object storage.

``` text
S3
 │
 └── Bucket
       ├── Object
       ├── Object
       └── Object
```

------------------------------------------------------------------------

# 32. S3 Bucket

A bucket is the top-level container for S3 objects.

Our bucket:

``` text
krb-enterprise-logs-app
```

Conceptually:

``` text
S3
 └── krb-enterprise-logs-app
       ├── krb-test.txt
       └── other objects
```

A bucket should not be thought of as a normal filesystem directory.

------------------------------------------------------------------------

# 33. S3 Object

An object is the actual data stored in S3.

Our test object:

``` text
krb-test.txt
```

It contained:

``` text
KRB Academy IAM test
```

An S3 object can have data, a key, metadata, and version information
when versioning is enabled.

------------------------------------------------------------------------

# 34. S3 Object Key

The key identifies the object within the bucket.

For our object:

``` text
Bucket:
krb-enterprise-logs-app

Key:
krb-test.txt
```

S3 URI:

``` text
s3://krb-enterprise-logs-app/krb-test.txt
```

Important:

`krb-test.txt` is the object key in this simple example.

If the key were:

``` text
logs/2026/09/krb-test.txt
```

the entire string would be the key.

The console may display prefixes as folders, but S3 fundamentally stores
objects identified by keys.

------------------------------------------------------------------------

# 35. Why S3 is Useful for KRB Enterprise

Instead of storing persistent files only on EC2:

``` text
EC2
 └── local files
```

we can use:

``` text
KRB Enterprise
      ↓
     S3
      ↓
Persistent objects
```

This separates compute from object storage.

If EC2 is terminated, S3 objects can remain.

------------------------------------------------------------------------

# 36. S3 Storage Classes --- Started

S3 offers different storage classes based on access patterns.

Basic mental model:

``` text
Frequently accessed
        ↓
     Standard

Rarely accessed
        ↓
Infrequent Access

Long-term archive
        ↓
     Glacier
```

Trade-offs include:

-   Storage cost
-   Access frequency
-   Retrieval characteristics/cost

Example:

``` text
Latest 7 days of logs
        ↓
Frequently accessed

Logs older than 1 year
        ↓
Rarely accessed / archive
```

We stopped before going deeper into the individual classes.

------------------------------------------------------------------------

# 37. Current KRB Academy AWS Progress

## Completed

### AWS Networking

-   Region
-   Availability Zones
-   High availability
-   VPC
-   Subnets
-   Public/private subnet concepts
-   Route Tables
-   Internet Gateway
-   NAT Gateway
-   Load Balancer concept
-   Security Groups
-   Backend/private networking
-   PostgreSQL networking

### EC2

-   EC2 fundamentals
-   AMI
-   Instance types
-   T/M/C/R families
-   CPU credits
-   Spot Instances
-   EBS
-   ENI
-   Private/public IP
-   DNS
-   Stop vs Terminate
-   SSH
-   Security Groups
-   Practical EC2 deployment
-   Windows SSH
-   iPad SSH
-   EBS persistence verification

### IAM

-   Authentication vs Authorization
-   Users
-   Groups
-   Roles
-   Policies
-   Least privilege
-   Effect
-   Action
-   Resource
-   Condition
-   Explicit Deny
-   Trust Policy
-   Permissions Policy
-   `sts:AssumeRole`
-   EC2 → IAM Role
-   Temporary credentials
-   EC2 → S3
-   Allowed vs denied permission testing

### S3

-   Object storage
-   Bucket
-   Object
-   Object key
-   S3 URI
-   S3 use with EC2/IAM
-   Storage classes introduced

------------------------------------------------------------------------

# 38. Next Session

Resume from:

``` text
S3 Storage Classes
        ↓
S3 Standard
        ↓
Infrequent Access
        ↓
Glacier / Archive
        ↓
Versioning
        ↓
Encryption
        ↓
Lifecycle Rules
        ↓
Bucket Policies
        ↓
Presigned URLs
```

------------------------------------------------------------------------

# 39. Core AWS Architecture Learned

``` text
                         AWS Region
                              │
                 ┌────────────┴────────────┐
                 │                         │
                AZ-1                      AZ-2
                 │                         │
                 └────────── VPC ──────────┘
                              │
                    ┌─────────┴─────────┐
                    │                   │
              Public Subnet       Private Subnet
                    │                   │
              Load Balancer          EC2
                    │                   │
                    │             IAM Role
                    │                   │
                    │                Backend
                    │                   │
                    │              PostgreSQL
                    │
                 Internet

EC2
 │
 └── EBS

EC2
 │
 └── IAM Role
       │
       └── S3 permissions
              │
              ▼
       S3 Bucket
              │
              └── Objects
```

# 40. Key Principles

1.  Use multiple Availability Zones for high availability.
2.  Keep backend/database resources private.
3.  Allow only required network paths and ports.
4.  Use EBS for EC2 block storage and S3 for object storage.
5.  Use IAM Roles for AWS workloads instead of embedding long-lived
    credentials when possible.
6.  Apply least privilege.
7.  Trust Policy = who can assume the role.
8.  Permissions Policy = what the role can do.
9.  Explicit Deny overrides Allow.
10. Separate compute from persistent object storage when appropriate.
