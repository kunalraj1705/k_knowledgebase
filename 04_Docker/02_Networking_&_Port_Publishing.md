# KRB Academy

# Docker — Networking & Port Publishing

**Session:** Docker Fundamentals  
**Date:** 26 August 2026  
**Project Context:** KRB Enterprise  
**Status:** Completed

---

## 1. Session Objective

Today's session continued the Docker fundamentals roadmap.

Topics covered:

- `CMD` vs `ENTRYPOINT`
- `.dockerignore`
- Docker build context
- Docker networking
- Container-to-container communication
- Docker DNS/service discovery
- Port publishing
- Host-to-container vs container-to-container communication

The goal was to understand these concepts before Dockerizing KRB Enterprise.

---

## 2. CMD vs ENTRYPOINT

### CMD

`CMD` defines the default command or default arguments for a container.

```dockerfile
FROM ubuntu:24.04

CMD ["echo", "Hello Docker"]
```

A normal command supplied to `docker run` can replace the `CMD`.

```bash
docker run --rm cmd-demo:v1 echo "Hello Kunal"
```

Mental model:

```text
CMD
 ↓
Default command / arguments
```

### ENTRYPOINT

`ENTRYPOINT` defines the executable that the container is built around.

```dockerfile
FROM ubuntu:24.04

ENTRYPOINT ["echo"]
```

Running:

```bash
docker run --rm entrypoint-demo:v1 "Hello Docker"
```

effectively executes:

```text
echo "Hello Docker"
```

Mental model:

```text
ENTRYPOINT
     ↓
Main executable
```

### ENTRYPOINT + CMD

They can be used together:

```dockerfile
FROM ubuntu:24.04

ENTRYPOINT ["echo"]

CMD ["Hello Docker"]
```

Without an argument, the effective command is:

```text
echo "Hello Docker"
```

With another argument, the default `CMD` argument is replaced.

Mental model:

```text
ENTRYPOINT + CMD
       ↓
Executable + default arguments
```

### Spring Boot connection

For KRB Enterprise, the eventual main container process will be the Spring Boot Java process:

```text
Container
    ↓
java -jar k_enterprise.jar
    ↓
Spring Boot application
```

The application itself should be the container's main process rather than using an artificial process simply to keep the container running.

---

## 3. `.dockerignore` and Build Context

When running:

```bash
docker build -t my-image:v1 .
```

the final `.` means the current directory is the Docker build context.

```text
Local Project
     ↓
Build Context
     ↓
Dockerfile / Build
     ↓
Image
```

Files inside the build context can be made available to Dockerfile instructions such as:

```dockerfile
COPY hello.txt .
```

A real project may contain:

```text
.git/
.idea/
.vscode/
target/
logs/
temporary files
secrets
large files
```

These may not be required for the image build.

### `.dockerignore`

`.dockerignore` specifies files and directories excluded from the Docker build context.

Example:

```text
.git
.idea
target
*.log
```

A learning example verified that:

```text
secret.txt
logs/
```

could be excluded.

Important distinction:

```text
.dockerignore
     ↓
Controls files included in Docker build context
```

It does not delete files from the local filesystem.

### `.dockerignore` vs `.gitignore`

```text
.gitignore
    ↓
Controls what Git tracks

.dockerignore
    ↓
Controls what Docker receives as build context
```

They are similar in concept but are not interchangeable.

---

## 4. Docker Networking

Docker networking allows containers to communicate with each other.

```text
┌──────────────────────────────────────────┐
│ Docker Network                           │
│                                          │
│  ┌───────────────┐    ┌───────────────┐ │
│  │ Container A   │───▶│ Container B   │ │
│  └───────────────┘    └───────────────┘ │
│                                          │
└──────────────────────────────────────────┘
```

### `localhost` inside a container

`localhost` inside a container refers to that container itself.

```text
Container A
    ↓
localhost
    ↓
Container A
```

It does not automatically refer to another container.

Therefore, if Spring Boot and PostgreSQL run in separate containers:

```text
localhost:5432
```

inside the Spring Boot container would refer to the Spring Boot container itself.

---

## 5. Creating a Docker Network

A user-defined network was created:

```bash
docker network create krb-network
```

Two Ubuntu containers were then created on that network:

```bash
docker run -dit --name container-a --network krb-network ubuntu:24.04

docker run -dit --name container-b --network krb-network ubuntu:24.04
```

The containers were verified using:

```bash
docker ps
```

---

## 6. Container-to-Container DNS

Docker provides DNS-based service discovery on user-defined networks.

From `container-a`:

```bash
docker exec -it container-a /bin/bash
```

Then:

```bash
getent hosts container-b
```

The container name resolved to the address of `container-b`.

This demonstrated:

```text
container-a
     │
     │ Docker DNS
     ▼
container-b
```

Applications can therefore use container/service names rather than hard-coded container IP addresses.

---

## 7. KRB Enterprise Networking Model

Eventually:

```text
┌─────────────────────────┐
│ KRB Enterprise          │
│ Spring Boot Container   │
│ :8282                   │
└────────────┬────────────┘
             │
        Docker Network
             │
        postgres:5432
             │
             ▼
┌─────────────────────────┐
│ PostgreSQL Container    │
│ :5432                   │
└─────────────────────────┘
```

The Spring Boot container can communicate with PostgreSQL through the Docker network using the PostgreSQL service/container name.

---

## 8. Port Publishing

Container-to-container networking is different from host-to-container access.

Docker publishes a container port using:

```bash
-p HOST_PORT:CONTAINER_PORT
```

Example:

```bash
docker run -d --name nginx-network-demo -p 8080:80 nginx:latest
```

This creates:

```text
Host :8080
     │
     │ Docker port mapping
     ▼
Container :80
```

The published port was verified with:

```bash
docker ps
docker port nginx-network-demo
```

Nginx was accessed from the host using:

```bash
curl http://localhost:8080
```

This verified:

```text
curl
  ↓
localhost:8080
  ↓
Ubuntu host
  ↓
Docker port mapping
  ↓
Nginx container :80
```

---

## 9. Host Port vs Container Port

Host and container ports do not need to be the same.

Example:

```bash
-p 9090:8282
```

means:

```text
Host :9090
     ↓
Container :8282
```

The application can listen on `8282` inside the container while the host exposes it on `9090`.

---

## 10. `EXPOSE` vs `-p`

### `EXPOSE`

```dockerfile
EXPOSE 8282
```

Documents the port the application expects to use inside the container.

It does not automatically publish the port to the host.

### `-p`

```bash
docker run -p 8282:8282 ...
```

Actually publishes the container port to the host.

Mental model:

```text
EXPOSE
    ↓
Container port metadata/documentation

-p
    ↓
Actual host → container port publishing
```

---

## 11. Host-to-Container vs Container-to-Container

### Host → Container

```text
Host
localhost:8282
     │
     │ -p 8282:8282
     ▼
Spring Boot container
:8282
```

### Container → Container

```text
Spring Boot container
        │
        │ Docker Network
        │ postgres:5432
        ▼
PostgreSQL container
```

The second path does not require PostgreSQL to publish port `5432` to the host.

A service does not need to be exposed to the host simply because another container needs to communicate with it.

---

## 12. Hands-on Work Completed

### CMD / ENTRYPOINT

- Built a CMD-based image.
- Ran the image.
- Overrode the default CMD.
- Built an ENTRYPOINT + CMD image.
- Supplied alternate arguments.
- Verified the behavior.

### `.dockerignore`

- Created a sample build context.
- Added files that should not be copied.
- Created `.dockerignore`.
- Rebuilt the image.
- Verified ignored files were excluded.

### Docker Networking

Created:

```bash
docker network create krb-network
```

Created two containers on the network:

```bash
docker run -dit --name container-a --network krb-network ubuntu:24.04

docker run -dit --name container-b --network krb-network ubuntu:24.04
```

Verified DNS:

```bash
docker exec -it container-a /bin/bash
```

then:

```bash
getent hosts container-b
```

### Port Publishing

Created:

```bash
docker run -d --name nginx-network-demo -p 8080:80 nginx:latest
```

Verified:

```bash
docker ps
docker port nginx-network-demo
curl http://localhost:8080
```

---

## 13. Key Mental Model

```text
                    Docker Host
                         │
                    localhost:8080
                         │
                         ▼
                ┌────────────────┐
                │ Nginx          │
                │ Container      │
                │ :80            │
                └────────────────┘
```

For multiple containers:

```text
                    Docker Network
                         │
             ┌───────────┴───────────┐
             │                       │
             ▼                       ▼
      Spring Boot               PostgreSQL
      :8282                     :5432
             │                       ▲
             └──── postgres:5432 ───┘
```

---

## 14. Session Completion

- [x] CMD
- [x] ENTRYPOINT
- [x] ENTRYPOINT + CMD
- [x] Command overriding
- [x] Spring Boot main-process concept
- [x] `.dockerignore`
- [x] Docker build context
- [x] User-defined Docker networks
- [x] Container-to-container communication
- [x] Docker DNS/service discovery
- [x] Container names as hostnames
- [x] `localhost` behavior inside containers
- [x] Port publishing
- [x] Host port vs container port
- [x] `EXPOSE` vs `-p`
- [x] Host-to-container communication
- [x] Container-to-container communication

---

## 15. KRB Enterprise Connection

We are still in the Docker fundamentals phase.

We have **not yet Dockerized KRB Enterprise**.

The eventual architecture will be:

```text
                 Host
                  │
             :8282 / API
                  │
                  ▼
        ┌──────────────────┐
        │ KRB Enterprise   │
        │ Spring Boot      │
        │ Container        │
        └────────┬─────────┘
                 │
           Docker Network
                 │
           postgres:5432
                 │
                 ▼
        ┌──────────────────┐
        │ PostgreSQL       │
        │ Container        │
        └──────────────────┘
```

Understanding networking first will make the later Docker Compose configuration easier to understand.

---

## 16. Next Session

The next Docker lesson is:

# Docker Volumes & Persistent Storage

We will learn:

```text
Container filesystem
        ↓
Ephemeral container data
        ↓
Docker volumes
        ↓
Bind mounts
        ↓
Volume lifecycle
        ↓
PostgreSQL persistence
```

Key question:

> What happens to PostgreSQL data when its container is deleted?

After volumes, we will continue with the remaining Docker fundamentals before Dockerizing KRB Enterprise.

---

## KRB Academy Progress

```text
Docker
│
├── Why Docker?                         ✅
├── Architecture                        ✅
├── Images & Containers                 ✅
├── Image Layers                        ✅
├── Tags, Digests & Registries          ✅
├── Container Lifecycle                 ✅
├── Dockerfile Fundamentals             ✅
├── docker build                        ✅
├── Build Context                       ✅
├── CMD vs ENTRYPOINT                   ✅
├── .dockerignore                       ✅
├── Docker Networking                   ✅
├── Container-to-Container DNS          ✅
├── Port Publishing                     ✅
├── Docker Volumes                      ⏭
├── Environment/Configuration           ⏭
├── Multi-stage Builds                  ⏭
├── Docker Compose                      ⏭
└── KRB Enterprise Dockerization       ⏭
```

---

**End of KRB Academy session — 26 August 2026**
