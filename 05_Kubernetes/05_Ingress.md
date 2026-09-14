# KRB Academy --- Kubernetes Ingress

## Date: 2026-09-14

### 1. Ingress

Kubernetes Ingress provides HTTP/HTTPS routing from external clients to
Services inside a cluster.

``` text
Client
  ↓
Ingress Controller
  ↓
Ingress Rules
  ↓
Service
  ↓
Pods
```

Ingress is primarily concerned with application-level HTTP/HTTPS routing
such as hostnames and paths.

### 2. Ingress Resource vs Ingress Controller

**Ingress Resource** - A Kubernetes API object containing routing
rules. - Defines which host/path should route to which Service. - Does
not itself implement network traffic handling.

**Ingress Controller** - Software that watches Ingress resources and
implements the routing. - Examples include ingress-nginx and Traefik. -
An Ingress resource alone does not provide working HTTP routing without
a controller.

### 3. Host-Based Routing

Ingress can route traffic using the HTTP `Host` header.

Example:

``` text
uat-customer.example.com → customer-service
uat-payment.example.com  → payment-service
uat-order.example.com    → order-service
```

This provides an application-specific hostname instead of requiring
path-based separation.

### 4. Ingress Routes to Services

Ingress routes to Kubernetes Services, not directly to Pods.

``` text
Ingress
   ↓
Service
   ↓
Pod replicas
```

The Service remains the stable networking abstraction while Pods can be
replaced, rescheduled, or scaled.

### 5. Ingress Controller Exposure

In a bare-metal or lab cluster, an Ingress Controller can be exposed
using a NodePort:

``` text
External Client
      ↓
Node IP : NodePort
      ↓
Ingress Controller
      ↓
Service
```

In production, a LoadBalancer or external load balancer is commonly
used.

### 6. Hostname Resolution

The hostname used by the client must resolve to a reachable address.

For local development/UAT, this can be provided by a hosts-file entry or
internal DNS.

Hostname resolution is separate from Ingress routing:

``` text
Hostname resolution
        ↓
Network reachability
        ↓
Ingress Controller
        ↓
Ingress routing
```

### 7. Path-Based vs Host-Based Routing

Path-based routing:

``` text
example.com/customer → customer-service
example.com/payment  → payment-service
```

Host-based routing:

``` text
customer.example.com → customer-service
payment.example.com  → payment-service
```

Host-based routing is useful when applications need independently
addressable hostnames.

### 8. Overall Model

``` text
Client
  ↓
Hostname / DNS
  ↓
Ingress Controller
  ↓
Ingress
  ↓
Service
  ↓
Pods
```

Key distinctions:

-   Deployment manages stateless application Pods.
-   Service provides a stable endpoint for Pods.
-   Ingress defines HTTP/HTTPS routing rules.
-   Ingress Controller implements those rules.
-   DNS/hosts resolution makes the hostname reachable by clients.

### 9. Interview Takeaways

-   Ingress is a Kubernetes API resource, not the traffic-processing
    implementation itself.
-   An Ingress Controller is required to implement Ingress rules.
-   Ingress routes HTTP/HTTPS traffic to Services.
-   Ingress should not directly target Pods.
-   Host-based routing allows different applications to use different
    hostnames.
-   NodePort is one way to expose an Ingress Controller in a
    lab/bare-metal environment.
-   DNS or hosts-file resolution is separate from Kubernetes Ingress
    routing.
