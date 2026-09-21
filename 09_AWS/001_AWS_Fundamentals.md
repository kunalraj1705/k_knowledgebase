# KRB Academy — AWS Fundamentals Notes
## Session: AWS Infrastructure & Networking Foundations
### Date: 2026-09-21

## 1. AWS Region
A Region is a geographical area containing multiple isolated Availability Zones.

Examples:
- Mumbai: `ap-south-1`
- Frankfurt: `eu-central-1`
- Ireland: `eu-west-1`
- Singapore: `ap-southeast-1`

Key idea:
> Region = geographical boundary containing multiple AZs.

Regions provide geographic separation for latency, data residency, regulatory requirements, and disaster recovery.

## 2. Availability Zones
An Availability Zone (AZ) is an isolated infrastructure location within a Region.

```text
AWS Region
├── AZ-A
├── AZ-B
└── AZ-C
```

Multiple AZs improve high availability. If one AZ becomes unavailable, workloads deployed in another AZ can continue serving requests.

Important:
> Multiple AZs do not automatically make an application highly available. The workload must actually be distributed across AZs.

## 3. VPC — Virtual Private Cloud
A VPC is a virtual network inside AWS.

It provides control over:
- IP address ranges
- Subnets
- Routing
- Internet connectivity
- Network isolation/security boundaries

Important correction:
> A VPC is Regional; it is not inside one AZ.

```text
Region
├── AZ-A → Subnets
└── AZ-B → Subnets
        \  VPC  /
```

## 4. VPC CIDR
Example VPC CIDR:

```text
10.0.0.0/16
```

Can be divided into smaller subnet ranges:

```text
10.0.1.0/24
10.0.2.0/24
10.0.3.0/24
```

Mental model:
> VPC = overall network; Subnet = smaller network segment inside the VPC.

## 5. Public vs Private Subnets
A subnet is public when its route table has a route to an Internet Gateway.

Typical architecture:

```text
Public Subnet
└── Load Balancer

Private Subnet
└── Backend

Private Subnet
└── Database
```

Public subnet does not mean every resource is automatically Internet-accessible. Reachability also depends on routing, public IPs, Security Groups, NACLs, and service configuration.

## 6. Route Tables
A route table directs network traffic based on destination.

Example:

```text
Destination        Target
10.0.0.0/16        local
0.0.0.0/0          Internet Gateway
```

`local` allows VPC-local traffic according to the applicable networking/security configuration.

`0.0.0.0/0` means any IPv4 destination that does not match a more specific route.

Important distinction:
> VPC route tables perform network/IP routing, not application routing.

For example:
- VPC routing: IP/network → next network target
- Application routing: `/login` → Login Service

Application routing is handled by components such as Load Balancers, API Gateways, Ingress, or application routers.

## 7. Internet Gateway
An Internet Gateway (IGW) provides a path between a VPC and the Internet.

A public subnet needs an appropriate route such as:

```text
0.0.0.0/0 → Internet Gateway
```

Attaching an IGW to the VPC alone does not make every resource public.

## 8. NAT Gateway
A NAT Gateway allows resources in private subnets to initiate outbound Internet connections without directly exposing those private resources to unsolicited inbound Internet connections.

```text
Private Backend
      ↓
Private Route Table
      ↓
NAT Gateway
      ↓
Internet Gateway
      ↓
Internet
```

The NAT Gateway is placed in a public subnet because it needs a path through the Internet Gateway.

## 9. IGW vs NAT Gateway

| Component | Purpose |
|---|---|
| Internet Gateway | Provides Internet connectivity for the VPC |
| NAT Gateway | Allows private resources to initiate outbound Internet connections |

## 10. Security Groups
A Security Group (SG) is a stateful, resource-level virtual firewall.

For KRB Enterprise:

```text
SG-LB
└── Inbound: 443 from Internet

SG-Backend
└── Inbound: 8080 from SG-LB

SG-Database
└── Inbound: 5432 from SG-Backend
```

Traffic flow:

```text
Internet
   ↓ HTTPS :443
Load Balancer
   ↓ HTTP :8080
Backend
   ↓ TCP :5432
PostgreSQL
```

Least-privilege principle:
- Do not allow Internet → Backend :8080
- Do not allow Internet → PostgreSQL :5432
- Allow Load Balancer → Backend :8080
- Allow Backend → PostgreSQL :5432

Important precision:
The backend normally uses an ephemeral source port when connecting to PostgreSQL. The relevant rule is the PostgreSQL destination port `5432`.

## 11. Security Group vs Network ACL

Security Group:
- Resource-level
- Stateful
- Allow rules

Network ACL:
- Subnet-level
- Stateless
- Supports allow and deny
- Inbound and outbound traffic must be considered separately

Mental model:
> NACL = subnet gate
> Security Group = resource gate

## 12. KRB Enterprise AWS Target Architecture

```text
                         Internet
                            │
                            │ HTTPS :443
                            ▼
                   Internet Gateway
                            │
                    Public Subnets
                            │
                      Load Balancer
                            │
                  ┌─────────┴─────────┐
                  ▼                   ▼
            Private Subnet A    Private Subnet B
                  │                   │
               Backend             Backend
                  │                   │
                  └─────────┬─────────┘
                            │
                       PostgreSQL
```

Outbound Internet from backend when required:

```text
Backend
   ↓
NAT Gateway
   ↓
Internet Gateway
   ↓
Internet
```

## 13. Core Mental Model

```text
Region
  ↓
Availability Zones
  ↓
VPC
  ↓
Subnets
  ↓
Route Tables
  ↓
Internet Gateway / NAT Gateway
  ↓
Security Groups
  ↓
Resources
```

Three routing layers:

```text
Application Routing
/login → Login Service

        ↓

Load Balancer / Ingress Routing
Client → Backend Target

        ↓

VPC Routing
IP/network → Next Network Target
```

## Interview Quick Revision

- **Region:** Geographical area containing multiple AZs.
- **AZ:** Isolated infrastructure location within a Region.
- **VPC:** Regional virtual network.
- **Subnet:** Network segment inside a VPC and associated with one AZ.
- **Public subnet:** Has a route to an Internet Gateway.
- **Route table:** Determines where network traffic is sent based on destination.
- **Internet Gateway:** Provides a path between VPC and Internet.
- **NAT Gateway:** Allows private resources to initiate outbound Internet connections without direct unsolicited inbound Internet access.
- **Security Group:** Stateful, resource-level firewall.
- **Network ACL:** Stateless, subnet-level firewall.

## Session Status

**AWS Fundamentals — Networking Foundations: COMPLETE**

Covered:
- Region
- Availability Zones
- VPC
- CIDR
- Subnets
- Public/private subnets
- Route tables
- Internet Gateway
- NAT Gateway
- Security Groups
- Network ACLs
- KRB Enterprise AWS network architecture

**Next session:** EC2 — compute, AMIs, instance types, EBS, public/private IPs, and connecting EC2 to the VPC architecture.
