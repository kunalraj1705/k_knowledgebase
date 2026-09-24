# KRB Academy --- AWS IAM Notes

## Session: IAM Fundamentals + EC2 Role

## 1. IAM Purpose

AWS IAM (Identity and Access Management) controls who can access AWS
resources and what they are allowed to do.

``` text
Authentication
    ↓
"Who are you?"

Authorization
    ↓
"Are you allowed to do this?"
```

For KRB Enterprise, authentication can identify a user through a JWT,
while authorization determines whether that user can perform the
requested operation.

``` text
401 → Authentication problem
403 → Authorization problem
```

## 2. IAM Core Concepts

``` text
IAM
├── Users
├── Groups
├── Policies
└── Roles
```

### IAM User

Represents a person or identity that can have long-term credentials.

### IAM Group

A collection of IAM users.

``` text
Developer 1 ─┐
Developer 2 ─┼── Developers Group ── Policy
Developer 3 ─┘
```

Groups make common permission management easier.

### IAM Role

A role is an identity that can be assumed.

For an EC2-hosted application:

``` text
EC2
 ↓
IAM Role
 ↓
AWS permissions
```

### IAM Policy

Defines permissions.

``` text
WHO
 ↓
CAN DO WHAT
 ↓
ON WHICH RESOURCE
```

## 3. Access Keys vs IAM Roles

Avoid embedding long-lived AWS access keys in an application when an IAM
role can be used.

Preferred workload pattern:

``` text
KRB Enterprise
      ↓
    EC2
      ↓
 IAM Role
      ↓
Temporary AWS credentials
      ↓
AWS service
```

The actual permissions of an access key depend on the IAM identity it
belongs to; an access key does not automatically provide access to an
entire AWS account or VPC.

Principle:

> Give an identity only the permissions it actually needs.

## 4. IAM Policy Structure

A policy commonly contains:

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
      "Resource": "arn:aws:s3:::krb-enterprise-logs/*"
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

Defines the operation.

``` text
s3:GetObject → read/download an object
s3:PutObject → upload/write an object
s3:DeleteObject → delete an object
```

If only `s3:PutObject` is allowed, the application can write but cannot
automatically delete.

### Resource

Defines where the permission applies.

``` text
arn:aws:s3:::krb-enterprise-logs/*
```

A specific resource scope is preferable to an unnecessarily broad `*`.

### Condition

Adds additional restrictions.

``` text
Allow
 ↓
Action
 ↓
Resource
 ↓
ONLY IF condition is satisfied
```

## 5. Explicit Deny

Important IAM rule:

> An explicit Deny overrides an Allow.

``` text
Allow + Explicit Deny
        ↓
      DENIED
```

## 6. Trust Policy vs Permissions Policy

This was the most important IAM concept covered today.

### Trust Policy

Answers:

> Who is allowed to assume this role?

### Permissions Policy

Answers:

> What can the role do after it is assumed?

Mental model:

``` text
Role
├── Trust Policy
│     ↓
│   WHO can assume?
│
└── Permissions Policy
      ↓
    WHAT can they do?
```

## 7. EC2 Trust Policy --- Hands-on

The role's trust policy was:

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

This does NOT grant S3 permissions. It establishes the trust
relationship.

## 8. KRB Enterprise IAM Architecture

``` text
              EC2
               │
               │ trusted
               ▼
      krb-academy-ec2-role
          /                    /             Trust Policy      Permissions Policy
     │                    │
     ▼                    ▼
EC2 can assume       s3:GetObject
the role             specific bucket
```

For the application:

``` text
KRB Enterprise on EC2
        ↓
IAM Role
        ↓
Limited IAM Policy
        ↓
S3
```

No long-lived AWS secret needs to be embedded in the application.

## 9. Planned Lab Policy

The planned limited policy is:

``` json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::krb-academy-logs/*"
    }
  ]
}
```

Planned names:

``` text
Role:
krb-academy-ec2-role

Policy:
krb-academy-s3-read
```

This grants read access to objects covered by the specified resource and
does not grant PutObject or DeleteObject unless separately allowed.

## 10. Practical Work Remaining

The IAM exercise is paused before the final test.

Next:

``` text
Create S3 bucket
       ↓
Upload test object
       ↓
Create/attach krb-academy-s3-read policy
       ↓
Attach IAM role to EC2
       ↓
SSH into EC2
       ↓
Test S3 access
       ↓
Verify allowed action
       ↓
Verify disallowed action
```

This will demonstrate the complete:

``` text
EC2
 ↓
IAM Role
 ↓
IAM Policy
 ↓
S3
```

flow.

## Current AWS Progress

### Completed

-   Region
-   Availability Zone
-   VPC
-   Subnet
-   Route Table
-   Internet Gateway
-   NAT Gateway
-   Load Balancer concept
-   Security Groups
-   Public vs private networking
-   EC2
-   AMI
-   Instance types
-   T / M / C / R families
-   vCPU / memory
-   CPU credits
-   T3 Unlimited
-   On-Demand / Savings Plans / Spot
-   EBS
-   ENI
-   Public/private IP
-   DNS
-   SSH
-   Stop vs Terminate
-   Practical EC2 launch
-   Public IP change after Stop/Start
-   EBS persistence
-   SSH from Windows and iPad
-   IAM authentication vs authorization
-   Users
-   Groups
-   Roles
-   Policies
-   Least privilege
-   Effect / Action / Resource / Condition
-   Explicit Deny
-   Trust Policy
-   Permissions Policy
-   EC2 IAM role creation started

### Next Session

Resume from:

``` text
IAM Role: krb-academy-ec2-role
        ↓
Create/attach S3 read policy
        ↓
Create test S3 bucket
        ↓
Attach role to EC2
        ↓
Test from EC2
```

## Key Takeaway

> Trust Policy answers "Who can assume this role?" Permissions Policy
> answers "What can the role do?" Least privilege means granting only
> the actions and resources the workload actually needs.
