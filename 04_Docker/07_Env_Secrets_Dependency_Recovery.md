# KRB Academy

# Docker — Compose Environment, Secrets, Multi-stage Builds & Runtime Dependency Recovery

**Session:** Docker Completion  
**Date:** 4 September 2026  
**Project Context:** KRB Enterprise  
**Status:** Completed

---

## 1. Session Objective

Complete the remaining Docker topics for KRB Academy by applying them to KRB Enterprise:

- Compose environment variables and `.env`
- Compose secrets
- Multi-stage Docker builds
- Runtime dependency recovery
- Final KRB Enterprise Dockerization verification

The objective was to complete the Docker learning track before moving to GitHub and CI/CD.

---

## 2. Compose Environment Variables

The KRB Enterprise Compose configuration originally contained runtime values directly:

```yaml
environment:
  SPRING_PROFILES_ACTIVE: uat
  DB_URL: jdbc:postgresql://postgres:5432/krb_enterprise
  DB_USERNAME: krb
  DB_PASSWORD: password
```

These values were changed to Compose variable references:

```yaml
environment:
  SPRING_PROFILES_ACTIVE: ${SPRING_PROFILES_ACTIVE}
  DB_URL: ${DB_URL}
  DB_USERNAME: ${DB_USERNAME}
  DB_PASSWORD: ${DB_PASSWORD}
```

A local `.env` file was introduced:

```env
SPRING_PROFILES_ACTIVE=uat
DB_URL=jdbc:postgresql://postgres:5432/krb_enterprise
DB_USERNAME=krb
DB_PASSWORD=password
```

The `.env` file is excluded from Git.

---

## 3. Compose Variable Substitution

Compose resolves `${VARIABLE}` references using the environment configuration.

Verification was performed with:

```bash
docker-compose config
```

The resolved configuration showed:

```yaml
DB_PASSWORD: password
DB_URL: jdbc:postgresql://postgres:5432/krb_enterprise
DB_USERNAME: krb
SPRING_PROFILES_ACTIVE: uat
```

This confirmed:

```text
.env
  ↓
Docker Compose
  ↓
Variable substitution
  ↓
Container environment
  ↓
Spring Boot
```

The complete KRB Enterprise stack was then started successfully.

---

## 4. Configuration vs Secrets

The distinction between configuration and secrets was established.

### Configuration

Examples:

```text
SPRING_PROFILES_ACTIVE
DB_URL
DB_USERNAME
```

### Secrets

Examples:

```text
DB_PASSWORD
JWT private key
API tokens
Client secrets
```

Important principle:

> Runtime configuration should be separated from the application image, and sensitive values should not be baked into the Docker image or committed to Git.

---

## 5. Compose Secrets

Docker Compose secrets were studied as a mechanism for making sensitive values available to a container as files.

Example:

```yaml
secrets:
  test_secret:
    file: ./secrets/test_secret.txt
```

and:

```yaml
services:
  krb-enterprise:
    secrets:
      - test_secret
```

Inside the container, the secret becomes available at:

```text
/run/secrets/test_secret
```

A temporary test secret was created, verified, and then removed. The stale Compose reference was also removed after the test file was deleted.

The final KRB Enterprise Compose configuration contains no `test_secret` reference.

---

## 6. JWT Key Handling

KRB Enterprise requires JWT signing keys.

The keys are currently mounted into the application container as read-only bind mounts:

```yaml
volumes:
  - /home/ubuntu/workspace/k_enterprise/secrets/private-key.pem:/root/.krb-enterprise/secrets/private-key.pem:ro
  - /home/ubuntu/workspace/k_enterprise/secrets/public-key.pem:/root/.krb-enterprise/secrets/public-key.pem:ro
```

The keys are not copied into the Docker image.

Current architecture:

```text
Host secret files
       ↓
Read-only bind mount
       ↓
KRB Enterprise container
       ↓
JWT security configuration
```

A production deployment can later replace this with a dedicated deployment secret mechanism.

---

## 7. Build Once, Configure at Runtime

A key deployment principle was established:

> Build once, configure at runtime.

The same application image should be usable across environments.

```text
                 Docker Image
                      │
              ┌───────┴───────┐
              ↓               ↓
             UAT             PROD
              │               │
       UAT configuration  PROD configuration
              │               │
          UAT secrets     PROD secrets
```

The application image should not contain environment-specific credentials.

---

## 8. Multi-stage Docker Builds

A multi-stage Dockerfile was implemented:

```dockerfile
FROM maven:3.9-eclipse-temurin-25 AS build

WORKDIR /build

COPY k_enterprise/pom.xml .
COPY k_enterprise/src ./src

RUN mvn clean package -DskipTests

FROM eclipse-temurin:25-jre

WORKDIR /app

COPY --from=build /build/target/k_enterprise-0.0.1-SNAPSHOT.jar app.jar

ENTRYPOINT ["java", "-jar", "app.jar"]
```

The architecture is:

```text
Build Stage
───────────
Maven
JDK
Source
   ↓
mvn clean package
   ↓
JAR
   ↓
Runtime Stage
─────────────
JRE
JAR
   ↓
Final image
```

The build stage is not included in the final runtime image.

---

## 9. Repository Separation

The application and infrastructure repositories remain separate.

```text
~/workspace/
├── k_enterprise/
│   └── Application source
│
└── k_enterprise_infra/
    └── Docker / Compose / infrastructure
```

For the multi-stage build, the application repository was made available as part of a temporary Docker build context.

This does not merge the repositories.

The principle is:

```text
Application repository
        ↓
Temporary build context
        ↓
Multi-stage Docker build
        ↓
Runtime image
```

Repository separation is therefore preserved.

---

## 10. Multi-stage Image Verification

The multi-stage image was built as:

```text
krb-enterprise:0.0.2
```

The image was inspected successfully.

Important runtime properties:

```text
Architecture: amd64
OS: linux
Working directory: /app
Entrypoint: java -jar app.jar
Java: 25.0.4
```

The reported image size was approximately:

```text
169.8 MB
```

The final image uses:

```dockerfile
FROM eclipse-temurin:25-jre
```

and copies only the generated application JAR from the build stage.

---

## 11. KRB Enterprise Running with Multi-stage Image

Compose was updated to use:

```yaml
image: krb-enterprise:0.0.2
```

The running containers were verified:

```text
compose-krb-enterprise-1
    IMAGE: krb-enterprise:0.0.2
    STATUS: Up
    PORT: 8282

compose-postgres-1
    IMAGE: postgres:18
    STATUS: Up (healthy)
```

The application logs confirmed successful startup.

The application was tested through:

```bash
curl -i http://localhost:8282
```

The expected response was:

```text
HTTP/1.1 401
```

This confirmed that the application container was reachable and Spring Security was processing requests.

---

## 12. Runtime Dependency Recovery

The important distinction was established:

```text
depends_on
    = startup ordering

healthcheck
    = health observation

restart policy
    = container process recovery

HikariCP
    = database connection management

Application
    = runtime behavior
```

`depends_on` does not continuously supervise the dependency after startup.

---

## 13. PostgreSQL Failure Test

PostgreSQL was stopped while KRB Enterprise remained running.

The application did not automatically stop just because PostgreSQL was unavailable.

HikariCP detected that existing database connections had become invalid.

The application logs showed:

```text
HikariPool-1 - Failed to validate connection
(This connection has been closed.)
```

and:

```text
Possibly consider using a shorter maxLifetime value.
```

This demonstrated:

```text
PostgreSQL unavailable
       ↓
Existing DB connections become invalid
       ↓
HikariCP validates connections
       ↓
HikariCP detects closed connections
```

The `maxLifetime` message was understood as a Hikari diagnostic suggestion, not a reason to blindly change the configuration after this artificial failure test.

---

## 14. PostgreSQL Recovery Test

PostgreSQL was started again and became healthy.

A real database-backed KRB Enterprise operation was then tested using the token-generation flow.

The request reached the database and attempted to find the user.

Because the development database was empty, the operation correctly failed with invalid email/password rather than because of a database connectivity failure.

The important result was:

```text
PostgreSQL
    ↓
Healthy again
    ↓
KRB Enterprise still running
    ↓
Generate Token
    ↓
Database user lookup
    ↓
Database connection successfully obtained
    ↓
User not found
    ↓
Expected authentication failure
```

KRB Enterprise did not need to be restarted to regain database connectivity.

This demonstrated runtime database connection recovery at the application/HikariCP level.

---

## 15. Runtime Recovery Architecture

```text
                    Docker
                      │
          ┌───────────┴───────────┐
          ↓                       ↓
     Healthcheck             Restart policy
          │                       │
          ↓                       ↓
  Health observation       Process recovery


                    Application
                         │
                         ↓
                      HikariCP
                         │
                         ↓
                    PostgreSQL
```

These mechanisms solve different problems and should not be treated as interchangeable.

---

## 16. Final KRB Enterprise Docker Architecture

```text
                    Host
                     │
              Docker Compose
                     │
        ┌────────────┴────────────┐
        ↓                         ↓
 KRB Enterprise              PostgreSQL
   0.0.2                     postgres:18
        │                         │
        │                         │
        └──── Compose network ────┘
                  │
                  ↓
             postgres:5432

KRB Enterprise
    │
    ├── UAT runtime configuration
    ├── JWT key read-only mounts
    ├── Health/dependency startup
    └── HikariCP database connections

PostgreSQL
    │
    └── Persistent named volume
```

---

## 17. Docker Completion Status

All planned Docker topics are now complete:

```text
Why Docker?                         Completed
Architecture                        Completed
Images & Containers                 Completed
Image Layers                        Completed
Tags, Digests & Registries          Completed
Container Lifecycle                 Completed
Dockerfile Fundamentals             Completed
docker build                        Completed
Build Context                       Completed
CMD vs ENTRYPOINT                   Completed
.dockerignore                       Completed
Docker Networking                   Completed
Container-to-Container DNS          Completed
Port Publishing                     Completed
Docker Volumes                      Completed
Bind Mounts                         Completed
Environment/Configuration           Completed
Docker Compose Fundamentals         Completed
Compose Networking                  Completed
Compose Volumes                     Completed
Compose Port Publishing             Completed
depends_on                          Completed
Health Checks                       Completed
Restart Policies                    Completed
Service Verification                Completed
Compose Environment & Secrets       Completed
Multi-stage Builds                  Completed
Runtime Dependency Recovery         Completed
KRB Enterprise Dockerization        Completed
```

---

## Key Takeaways

1. Compose environment variables allow runtime configuration without hardcoding values directly into the service definition.

2. `.env` is useful for local configuration, but it is not a production secret-management system.

3. Secrets should not be baked into Docker images or committed to Git.

4. Multi-stage builds separate build tooling from the final runtime image.

5. The KRB Enterprise application and infrastructure repositories can remain separate while still supporting a multi-stage Docker build.

6. `depends_on` controls startup ordering; it is not continuous runtime dependency supervision.

7. Healthchecks observe health; restart policies handle container process termination.

8. HikariCP manages database connection pooling and can detect invalid connections and obtain usable connections again when PostgreSQL becomes available.

9. A database failure does not necessarily require restarting the entire application container.

10. The preferred deployment principle is:

```text
Build once
    ↓
Same image
    ↓
Configure at runtime
```

---

## Session Completion

```text
Compose Environment Variables       ✅
.env                                ✅
Compose Secrets                     ✅
Multi-stage Builds                  ✅
Runtime Dependency Recovery         ✅
KRB Enterprise Dockerization        ✅
```

**Docker Academy track: COMPLETE**

---

## Next Session

Move to:

# GitHub → CI/CD

Planned progression:

```text
GitHub
   ↓
CI
   ↓
Build + Test
   ↓
Docker Image
   ↓
Container Registry
   ↓
Deployment
   ↓
UAT
```

The application and infrastructure repositories will remain separate.

---

## KRB Academy Progress

```text
Docker
├── Fundamentals                    Completed
├── Dockerfile                      Completed
├── Networking                      Completed
├── Storage                         Completed
├── Docker Compose                  Completed
├── Environment & Secrets           Completed
├── Multi-stage Builds              Completed
├── Runtime Dependency Recovery     Completed
└── KRB Enterprise Dockerization    Completed

Next:
GitHub / CI/CD
```

---

**End of Session**
