# KRB Academy --- AWS EC2 Notes

## Session: 23 September 2026

## 1. EC2 Instance Types

An EC2 instance type defines compute characteristics such as vCPU,
memory, network performance and other capabilities.

  Family   Primary focus
  -------- -------------------
  T        Burstable
  M        General purpose
  C        Compute optimized
  R        Memory optimized
  I        Storage optimized
  G / P    Accelerated / GPU

### T --- Burstable

Use when CPU is normally low/moderate but occasionally spikes.

``` text
Normal usage → CPU credits accumulate → spike → credits consumed → burst
```

### M --- General Purpose

Balanced CPU and memory. Common for Spring Boot applications, backend
APIs, microservices and application servers.

### C --- Compute Optimized

For predominantly CPU-bound workloads.

### R --- Memory Optimized

For memory-intensive workloads.

Memory trick:

``` text
T → Temporary/occasional CPU spikes
M → Mixed/balanced
C → CPU heavy
R → RAM heavy
```

## 2. Choosing an Instance

Do not choose the biggest instance by default.

``` text
Application
  ↓
Measure CPU / memory / network / latency / traffic
  ↓
Choose family
  ↓
Choose size
  ↓
Monitor
  ↓
Adjust
```

Vertical scaling:

``` text
8 vCPU → 16 vCPU
```

Horizontal scaling:

``` text
EC2-A ─┐
EC2-B ─┼── Load Balancer
EC2-C ─┘
```

For a stateless Spring Boot backend, horizontal scaling can be
appropriate when traffic increases. Do not rely only on CPU; consider
memory, request count, latency and application metrics.

## 3. vCPU and Memory

A vCPU is a virtual CPU thread presented to the EC2 instance. More vCPUs
provide more potential CPU-level parallelism, but more vCPUs do not
automatically make an application faster.

The application must have enough concurrent/CPU-bound work to use the
additional capacity.

## 4. T-Series CPU Credits

T-series instances use CPU credits for burst performance.

``` text
Moderate baseline usage
        ↓
Credits accumulate
        ↓
CPU spike
        ↓
Credits consumed
        ↓
Burst CPU performance
```

T-series is suitable for occasional CPU bursts, not sustained high CPU
workloads.

### T3 Unlimited Mode

The lab instance showed:

``` text
Credit specification: unlimited
```

Unlimited mode allows a T3 instance to continue bursting after accrued
credits are depleted. Sustained CPU usage above baseline can result in
additional charges for surplus CPU usage.

CPU credits are performance capacity, not a money balance.

## 5. EC2 Pricing

### On-Demand

Pay for usage without long-term commitment. Useful for development,
testing, short-term and unpredictable workloads.

### Savings Plans / Reserved capacity

Commit to predictable usage for a period in exchange for lower effective
cost. Trade-off is lower cost versus reduced flexibility.

### Spot

Uses spare AWS capacity at lower cost, but AWS can reclaim the capacity.

Good for batch processing, data processing, CI workers and
fault-tolerant/disposable workloads.

Not appropriate as the sole dependency for a workload that cannot
tolerate interruption.

### Dedicated Hosts / Dedicated Instances

Conceptually relevant for licensing, compliance and specific isolation
requirements.

## 6. AWS Cost Protection

Created:

``` text
Budget: KRB-Academy-Monthly-Budget
Amount: $5.00
```

The budget is an alert mechanism, not a hard spending limit. The
template alerts at 85% actual spend, 100% actual spend, and forecasted
spend reaching 100%.

## 7. Practical EC2 Launch

Created:

``` text
Name: krb-academy-ec2
AMI: Amazon Linux 2023
Architecture: 64-bit x86
Instance type: t3.micro
Root EBS: 8 GiB gp3
Key pair: krb-academy-ec2-key
Public IPv4: enabled
SSH: TCP 22 from My IP
HTTP: disabled
HTTPS: disabled
```

Instance:

``` text
Instance ID: i-066a675366b3a14f6
Region: ap-southeast-2
Availability Zone: ap-southeast-2b
```

## 8. SSH

The Windows SSH key initially failed because the `.pem` file permissions
were too broad.

After removing broad ACL permissions and granting the local Windows user
read access, SSH succeeded.

Connection used:

``` bash
ssh -i ".\krb-academy-ec2-key.pem" ec2-user@ec2-3-26-78-210.ap-southeast-2.compute.amazonaws.com
```

Lesson:

> Private SSH keys must not be accessible to broad users/groups.

## 9. EC2 Networking

Actual instance networking:

``` text
VPC: vpc-03eca9a0c4e8e1ece
Subnet: subnet-0eb0ba847e757f92a
AZ: ap-southeast-2b
Private IPv4: 172.31.42.113
Public IPv4: 3.26.78.210
ENI: eni-00f45df89744c54
```

Mental model:

``` text
VPC
 ↓
Subnet
 ↓
ENI
 ├── Private IP
 └── Public IP association
      ↓
     EC2
```

The ENI is the instance's virtual network interface.

The instance had no Elastic IP, so its temporary public IPv4 can change
after Stop/Start.

## 10. DNS

Public DNS:

``` text
ec2-3-26-78-210.ap-southeast-2.compute.amazonaws.com
```

Private DNS:

``` text
ip-172-31-42-113.ap-southeast-2.compute.internal
```

DNS resolves a hostname to an endpoint. A Load Balancer, not DNS itself,
selects a backend target.

## 11. EBS

Configured root volume:

``` text
8 GiB gp3
```

Inside Linux:

``` text
/dev/nvme0n1p1 → /
```

Observed:

``` text
Size: 8.0G
Used: 1.7G
Available: 6.4G
```

Mental model:

``` text
EC2
 ↓
EBS
 ↓
Linux block device
 ↓
Filesystem
```

## 12. Linux Verification

Commands used:

``` bash
whoami
hostname
uname -a
df -h
free -h
```

Observed:

``` text
User: ec2-user
OS: Amazon Linux 2023
Kernel: 6.18.x
vCPU: 2
Usable memory: ~913 MiB
Root filesystem: 8 GiB
Swap: 0
```

## 13. Stop vs Terminate

The instance was stopped at the end of the session.

### Stop

``` text
Running → Stopped
```

-   Compute stops.
-   EBS root volume remains.
-   Data persists on EBS.
-   Temporary public IPv4 is released.
-   Starting again can assign a different public IPv4.

### Terminate

``` text
Running → Terminated
```

-   Instance is permanently removed.
-   Root EBS is normally deleted when DeleteOnTermination is enabled.
-   The instance cannot be started again.

Key principle:

> Stop is reversible. Terminate is destructive.

## 14. EC2 Architecture

``` text
AWS Region
  ↓
Availability Zone
  ↓
VPC
  ↓
Subnet
  ↓
ENI
  ├── Private IP
  └── Public IP association
       ↓
      EC2
       ├── vCPU
       ├── Memory
       └── EBS
```

## 15. Current Status

Completed:

-   EC2 instance types
-   T / M / C / R families
-   vCPU and memory
-   CPU credits
-   T3 Unlimited mode
-   EC2 pricing models
-   AWS budget
-   Practical EC2 launch
-   AMI
-   Key pair
-   Security Group
-   VPC / subnet / AZ
-   Public/private IP
-   ENI
-   DNS
-   EBS
-   SSH
-   Linux verification
-   Stop vs Terminate

Current instance state:

``` text
krb-academy-ec2
State: STOPPED
AZ: ap-southeast-2b
```

### Next session

``` text
STOPPED
  ↓
START
  ↓
Check public IP change
  ↓
Verify EBS persistence
  ↓
SSH again
  ↓
Terminate
  ↓
EC2 module COMPLETE
```

Then continue to the next AWS module.

## Key Takeaway

> EC2 is compute. The instance type determines compute characteristics,
> the ENI provides networking, the subnet/VPC determine network
> placement, the Security Group controls traffic, and EBS provides
> persistent block storage.
