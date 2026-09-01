# KRB Academy — Docker Compose

**Session:** Docker Fundamentals\
**Date:** 1 September 2026\
**Project Context:** KRB Enterprise\
**Status:** Completed

------------------------------------------------------------------------

## 1. Session Objective

Today's session focused on Docker Compose and multi-container
application architecture.

Topics covered:

-   Docker Compose services
-   Compose project naming
-   Automatic networks
-   Docker DNS and service names
-   Internal service communication
-   Named volumes
-   PostgreSQL persistence
-   PostgreSQL 18 volume mount path
-   Port publishing
-   Host-to-container vs container-to-container communication
-   Internal-only database design

------------------------------------------------------------------------

## 2. What Is Docker Compose?

Docker Compose is used to define and run multiple related containers
using a single configuration file:

``` text
compose.yaml
```

Conceptually:

``` text
compose.yaml
      ↓
Docker Compose
      ↓
Services
      ├── Application
      └── Database
```

------------------------------------------------------------------------

## 3. Services and Project Naming

The learning project directory was:

``` text
~/docker-learning/compose-demo
```

Compose used the project name:

``` text
compose-demo
```

to create resources such as:

``` text
compose-demo-postgres-1
compose-demo-ubuntu-1
compose-demo_default
compose-demo_postgres_data
```

Important distinction:

``` text
Service name    → postgres
Container name  → compose-demo-postgres-1
```

------------------------------------------------------------------------

## 4. Starting and Stopping Compose

Foreground mode:

``` bash
docker-compose up
```

Detached mode:

``` bash
docker-compose up -d
```

Check running containers:

``` bash
docker ps
```

Stop and remove Compose-managed containers and the default network:

``` bash
docker-compose down
```

Important:

``` text
docker-compose down
        ↓
Containers removed
Networks removed
Named volumes remain
```

To remove volumes as well:

``` bash
docker-compose down -v
```

This can delete persistent database data.

------------------------------------------------------------------------

## 5. PostgreSQL Service Configuration

The PostgreSQL service used:

``` yaml
services:
  postgres:
    image: postgres:18
    environment:
      POSTGRES_DB: krb_enterprise
      POSTGRES_USER: krb
      POSTGRES_PASSWORD: password
```

These variables configure the PostgreSQL image during first-time
initialization.

``` text
POSTGRES_DB       → krb_enterprise database
POSTGRES_USER     → krb user
POSTGRES_PASSWORD → user password
```

They are initialization/configuration values, not application connection
commands.

------------------------------------------------------------------------

## 6. Automatic Compose Network

Docker Compose automatically created:

``` text
compose-demo_default
```

The network driver was:

``` text
bridge
```

Conceptually:

``` text
compose-demo_default
│
├── postgres
│
└── ubuntu
```

The network was inspected using:

``` bash
docker network inspect compose-demo_default
```

The PostgreSQL container had an internal Docker IP such as:

``` text
172.19.0.2
```

------------------------------------------------------------------------

## 7. Docker DNS and Service Discovery

Inside the Ubuntu container:

``` bash
getent hosts postgres
```

resolved:

``` text
postgres → 172.19.0.2
```

Conceptually:

``` text
Ubuntu container
      │
      │ postgres
      ▼
Docker internal DNS
      │
      ▼
PostgreSQL container
```

Important principle:

``` text
Use service names
        ↓
Do not depend on container IP addresses
```

The future KRB Enterprise database URL inside Compose will use:

``` text
jdbc:postgresql://postgres:5432/krb_enterprise
```

------------------------------------------------------------------------

## 8. Why Not Use Container IP Addresses?

Container IP addresses can change when containers are recreated.

The service name remains stable:

``` text
postgres
```

Therefore the application should use:

``` text
postgres:5432
```

instead of a hard-coded container IP.

------------------------------------------------------------------------

## 9. Internal Container Connectivity

A temporary Ubuntu service was used as a test container:

``` yaml
ubuntu:
  image: ubuntu:24.04
  command: sleep infinity
```

`command: sleep infinity` keeps the test container running.

We entered it using:

``` bash
docker exec -it compose-demo-ubuntu-1 bash
```

DNS was tested with:

``` bash
getent hosts postgres
```

PostgreSQL connectivity was tested with:

``` bash
nc -zv postgres 5432
```

Result:

``` text
Connection to postgres (...) 5432 port succeeded!
```

This proved:

``` text
Ubuntu container
      │
      │ postgres:5432
      ▼
PostgreSQL container
```

------------------------------------------------------------------------

## 10. Internal Service Communication

Services inside the same Compose network communicate using:

``` text
SERVICE_NAME:CONTAINER_PORT
```

For PostgreSQL:

``` text
postgres:5432
```

For KRB Enterprise:

``` properties
spring.datasource.url=jdbc:postgresql://postgres:5432/krb_enterprise
```

Inside Docker, `localhost` refers to the current container itself, not
the PostgreSQL container.

------------------------------------------------------------------------

## 11. Docker Compose Named Volumes

A named volume was declared:

``` yaml
volumes:
  postgres_data:
```

Compose created:

``` text
compose-demo_postgres_data
```

Pattern:

``` text
<Project Name>_<Volume Name>
```

------------------------------------------------------------------------

## 12. PostgreSQL 18 Volume Mount Path

An important practical issue was encountered.

Using:

``` text
/var/lib/postgresql/data
```

caused a PostgreSQL 18 volume-layout error.

The corrected mount for this setup was:

``` yaml
volumes:
  - postgres_data:/var/lib/postgresql
```

Conceptually:

``` text
compose-demo_postgres_data
        ↓
/var/lib/postgresql
        ↓
PostgreSQL version-specific data directory
```

------------------------------------------------------------------------

## 13. Volume Persistence

A test table was created:

``` sql
CREATE TABLE docker_test (
    id INT PRIMARY KEY,
    name VARCHAR(100)
);
```

Test data:

``` sql
INSERT INTO docker_test (id, name)
VALUES (1, 'KRB Enterprise');
```

Verification:

``` sql
SELECT * FROM docker_test;
```

Result:

``` text
 id | name
----+----------------
  1 | KRB Enterprise
```

The services were removed using:

``` bash
docker-compose down
```

The containers and network were removed, but:

``` text
compose-demo_postgres_data
```

remained.

After starting Compose again, the same table and row still existed.

This proved:

``` text
Container removed
       ↓
Volume remains
       ↓
Container recreated
       ↓
Same volume attached
       ↓
Database data remains
```

Key principle:

``` text
Container = replaceable runtime
Volume    = persistent storage
```

------------------------------------------------------------------------

## 14. Port Publishing

PostgreSQL was temporarily published using:

``` yaml
ports:
  - "5432:5432"
```

Format:

``` text
HOST_PORT:CONTAINER_PORT
```

This creates:

``` text
Ubuntu Host:5432
        ↓
Docker port mapping
        ↓
PostgreSQL container:5432
```

`docker ps` showed a mapping similar to:

``` text
0.0.0.0:5432->5432/tcp
```

Host connectivity was verified using:

``` bash
nc -zv localhost 5432
```

The connection succeeded.

------------------------------------------------------------------------

## 15. Internal Port vs Published Port

### From another Compose container

Use:

``` text
postgres:5432
```

### From the Ubuntu host

Use:

``` text
localhost:5432
```

The two paths are different:

``` text
Inside Docker:

Application
    │
    ▼
postgres:5432
    │
    ▼
PostgreSQL


From Ubuntu Host:

localhost:5432
    │
    ▼
Docker Port Mapping
    │
    ▼
PostgreSQL:5432
```

------------------------------------------------------------------------

## 16. Why PostgreSQL Usually Should Remain Internal

For the final KRB Enterprise deployment:

``` text
External Client
      │
      ▼
KRB Enterprise Application
      │
      │ postgres:5432
      ▼
PostgreSQL
      │
      ▼
Persistent Volume
```

Because the application and database are already on the same Compose
network, PostgreSQL does not need a published host port when only the
application needs access.

The preferred internal connection is:

``` text
postgres:5432
```

------------------------------------------------------------------------

## 17. Current Learning Compose Configuration

The learning setup was conceptually:

``` yaml
services:
  postgres:
    image: postgres:18
    environment:
      POSTGRES_DB: krb_enterprise
      POSTGRES_USER: username
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql

  ubuntu:
    image: ubuntu:24.04
    command: sleep infinity

volumes:
  postgres_data:
```

The Ubuntu service was only a temporary learning/test container and will
not be part of the final KRB Enterprise deployment.

------------------------------------------------------------------------

## 18. KRB Enterprise Connection

Target architecture:

``` text
                 External Client
                        │
                        ▼
                  Host Port 8282
                        │
                        ▼
              ┌──────────────────┐
              │ KRB Enterprise   │
              │ Spring Boot      │
              │ Container        │
              └────────┬─────────┘
                       │
                 postgres:5432
                       │
                       ▼
              ┌──────────────────┐
              │ PostgreSQL       │
              │ Container        │
              └────────┬─────────┘
                       │
                       ▼
                Named Docker Volume
                       │
                       ▼
                Persistent DB Data
```

------------------------------------------------------------------------

## 19. Key Takeaways

1.  Docker Compose manages multiple related containers from one
    configuration file.
2.  Service names are used for internal service discovery.
3.  Docker Compose automatically creates a default network.
4.  Services communicate through Docker DNS.
5.  Container IP addresses should not be used as stable application
    configuration.
6.  Internal communication uses `service-name:container-port`.
7.  Named volumes persist independently from container recreation.
8.  `docker-compose down` keeps named volumes by default.
9.  `docker-compose down -v` can remove persistent volumes.
10. PostgreSQL 18 requires the correct volume mount layout.
11. Port publishing uses `HOST_PORT:CONTAINER_PORT`.
12. `localhost:5432` and `postgres:5432` are different communication
    paths.
13. PostgreSQL does not need a published host port when only internal
    Compose services need access.
14. KRB Enterprise should connect to PostgreSQL using the Compose
    service name.

------------------------------------------------------------------------

## 20. Session Completion

### Completed today

-   [x] Docker Compose fundamentals
-   [x] Compose services
-   [x] Compose project resource naming
-   [x] `docker-compose up`
-   [x] Detached mode with `-d`
-   [x] `docker-compose down`
-   [x] Automatic Compose network
-   [x] Bridge networking
-   [x] Docker DNS/service discovery
-   [x] Service-name resolution
-   [x] Container-to-container communication
-   [x] PostgreSQL connectivity test
-   [x] Compose named volumes
-   [x] Compose volume inspection
-   [x] PostgreSQL 18 volume mount correction
-   [x] PostgreSQL data persistence
-   [x] Container recreation with persistent data
-   [x] `down` vs `down -v`
-   [x] Port publishing
-   [x] Host-to-container communication
-   [x] Internal-only database communication
-   [x] Final KRB Enterprise Compose architecture direction

------------------------------------------------------------------------

## 21. Next Session

Next topics:

``` text
depends_on
        ↓
Service startup dependencies
        ↓
Health checks
        ↓
Database readiness
        ↓
Environment variables in Compose
        ↓
.env files and variable substitution
        ↓
Secrets for deployment
        ↓
Restart policies
        ↓
build vs image
        ↓
Final KRB Enterprise Compose setup
```

Important distinction:

``` text
Container started
        ≠
Application ready
```

The immediate next topic is:

# Docker Compose `depends_on`

followed by:

# Health Checks and Database Readiness

------------------------------------------------------------------------

## KRB Academy Progress

``` text
Docker
│
├── Why Docker?                         Completed
├── Architecture                        Completed
├── Images & Containers                 Completed
├── Image Layers                        Completed
├── Tags, Digests & Registries          Completed
├── Container Lifecycle                 Completed
├── Dockerfile Fundamentals             Completed
├── docker build                        Completed
├── Build Context                       Completed
├── CMD vs ENTRYPOINT                   Completed
├── .dockerignore                       Completed
├── Docker Networking                   Completed
├── Container-to-Container DNS          Completed
├── Port Publishing                     Completed
├── Docker Volumes                      Completed
├── Bind Mounts                         Completed
├── Environment/Configuration           Completed
├── Docker Compose Fundamentals         Completed
├── Compose Networking                  Completed
├── Compose Volumes                     Completed
├── Compose Port Publishing             Completed
├── depends_on                          Next
├── Health Checks                       Next
├── Restart Policies                    Upcoming
├── Compose Environment/Secrets         Upcoming
├── Multi-stage Builds                  Upcoming
└── KRB Enterprise Dockerization        Upcoming
```

------------------------------------------------------------------------

**End of KRB Academy session --- 1 September 2026**
