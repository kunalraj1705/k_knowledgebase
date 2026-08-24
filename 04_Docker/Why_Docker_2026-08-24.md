# KRB Academy

# Docker — Session Notes

**Session:** Docker Fundamentals  
**Date:** 24 August 2026  
**Project Context:** KRB Enterprise  
**Status:** Completed

---

## 1. Learning Objective

The goal of this session was to begin learning Docker before deploying KRB Enterprise.

Roadmap:

```text
Docker Fundamentals
        ↓
Docker Hands-on
        ↓
Dockerize KRB Enterprise
        ↓
Docker Compose
        ↓
Kubernetes Fundamentals
        ↓
Deploy KRB Enterprise to Kubernetes
```

The purpose is to understand Docker concepts first and then apply them to the real KRB Enterprise application.

---

## 2. Why Docker?

Without containers, an application depends heavily on the environment where it runs:

```text
Server
├── Operating System
├── Java version
├── Maven/build environment
├── PostgreSQL version
├── Libraries
├── Environment configuration
└── Application
```

Different servers can have different versions and configurations.

Docker provides a standardized way to package and run applications:

```text
Application
    +
Runtime
    +
Dependencies
    ↓
Docker Image
    ↓
Container
```

---

## 3. Docker Mental Model

The fundamental Docker flow is:

```text
Dockerfile
    │
    │ docker build
    ▼
  Image
    │
    │ docker run
    ▼
Container
    │
    ▼
Running Application
```

### Docker Image

A packaged, read-only template used to create containers.

Example:

```text
nginx:latest
```

### Container

A running instance created from an image.

```text
nginx:latest
      ↓
docker run
      ↓
nginx container
```

### Docker Engine

The system responsible for managing Docker resources such as images, containers, networks, and volumes.

### Docker CLI

The command-line interface used to interact with Docker.

```text
User
  ↓
Docker CLI
  ↓
Docker Engine
  ↓
Containers / Images / Networks / Volumes
```

---

## 4. Docker vs Virtual Machines

Traditional VM model:

```text
Physical Server
      ↓
Hypervisor
      ↓
Virtual Machine
      ↓
Guest Operating System
      ↓
Application
```

Containers generally work differently:

```text
Physical Server
      ↓
Operating System
      ↓
Docker Engine
      ↓
Containers
      ↓
Applications
```

Containers share the host kernel instead of each requiring a complete guest operating system.

---

## 5. Docker Images

An image is a packaged filesystem and metadata used to create containers.

Examples:

```text
ubuntu:24.04
nginx:latest
postgres:18
```

Eventually, KRB Enterprise will have its own application image, for example:

```text
krb-enterprise:1.0
```

An image itself is not a running application.

```text
Image
  ↓
docker run
  ↓
Container
```

---

## 6. Image Layers

Docker images are built using layers.

Conceptually:

```text
┌──────────────────────────────┐
│ Application files            │
├──────────────────────────────┤
│ Runtime / dependencies       │
├──────────────────────────────┤
│ Base image layers            │
├──────────────────────────────┤
│ Base filesystem              │
└──────────────────────────────┘
```

Unchanged layers can be reused between builds.

During the Nginx image pull, Docker showed layers as:

```text
Already exists
Pull complete
Pull complete
...
```

This demonstrated that Docker retrieves and reuses individual image layers.

---

## 7. Base Images

A Dockerfile commonly starts with a base image:

```dockerfile
FROM ubuntu:24.04
```

Conceptually:

```text
Base Image
    ↓
Dockerfile instructions
    ↓
New Image
```

For the future Spring Boot application image, an appropriate Java runtime base image will be selected.

---

## 8. Image Tags

Images commonly use:

```text
image:tag
```

Examples:

```text
nginx:latest
postgres:18
krb-enterprise:1.0
```

For:

```text
krb-enterprise:1.0
```

```text
Image name → krb-enterprise
Tag        → 1.0
```

`latest` is a tag and should not automatically be interpreted as the numerically newest version.

---

## 9. Image Digests

Docker can identify image content using a digest:

```text
sha256:...
```

The Nginx pull displayed an image digest.

Conceptually:

```text
Tag
 ↓
Image
 ↓
Content Digest
```

A digest identifies the image content more precisely than a tag.

---

## 10. Docker Registry

A registry stores and distributes Docker images.

```text
Developer
    │
    │ docker push
    ▼
Docker Registry
    │
    │ docker pull
    ▼
Server / Kubernetes
```

Images can be pulled from a registry or built locally.

This will eventually connect the Docker workflow to Kubernetes deployment.

---

## 11. Docker Installation Verification

Docker was verified on Ubuntu:

```bash
docker --version
```

Result:

```text
Docker version 29.1.3, build 29.1.3-0ubuntu4.1
```

Docker is installed and accessible.

---

## 12. Existing Docker Images

The environment already contained:

```text
cache-lab:v1
hello-world:latest
nginx:latest
postgres:18
testcontainers/ryuk:0.14.0
ubuntu:latest
```

The PostgreSQL image is relevant to KRB Enterprise, while the Testcontainers Ryuk image is related to integration testing.

---

## 13. Hands-on: Pulling an Image

Executed:

```bash
docker pull nginx:latest
```

Docker reported multiple layers as:

```text
Already exists
Pull complete
```

and returned an image digest.

The image was checked using:

```bash
docker images
```

Flow:

```text
Docker Registry
      ↓
docker pull
      ↓
Local Image
```

---

## 14. Hands-on: Running a Container

An Nginx container was created from the image.

Conceptually:

```text
nginx:latest
      ↓
docker run
      ↓
nginx-demo
      ↓
Running Container
```

The container was checked with:

```bash
docker ps
```

---

## 15. Container Inspection

The container was inspected using:

```bash
docker inspect nginx-demo
```

This demonstrated that Docker maintains metadata describing the container, including runtime configuration and networking information.

---

## 16. Container Logs

Container output can be inspected using:

```bash
docker logs nginx-demo
```

Logs remain available after a container has stopped.

---

## 17. Executing Commands Inside a Container

A shell was opened inside the Nginx container:

```bash
docker exec -it nginx-demo /bin/sh
```

Inside the container, filesystem locations such as:

```text
/etc/nginx
```

were explored.

The shell was exited using:

```bash
exit
```

This demonstrated that a container has its own filesystem and process environment.

---

## 18. Container Lifecycle

The basic lifecycle was demonstrated:

```text
create / run
    ↓
running
    ↓
stop
    ↓
stopped
    ↓
start
    ↓
running
    ↓
remove
```

Important commands:

```bash
docker ps
docker ps -a
docker start
docker stop
docker rm
```

`docker ps` shows running containers.

`docker ps -a` shows running and stopped containers.

Stopping a container does not automatically delete it.

---

## 19. Dockerfile

A Dockerfile is a text file containing instructions used to build a Docker image.

The first custom image used:

```dockerfile
FROM ubuntu:24.04

WORKDIR /app

COPY hello.txt .

CMD ["cat", "hello.txt"]
```

This was a learning exercise only. KRB Enterprise was not Dockerized yet.

---

## 20. Dockerfile — `FROM`

```dockerfile
FROM ubuntu:24.04
```

Defines the base image.

```text
ubuntu:24.04
      ↓
New Docker image
```

---

## 21. Dockerfile — `WORKDIR`

```dockerfile
WORKDIR /app
```

Sets the working directory inside the image/container.

---

## 22. Dockerfile — `COPY`

```dockerfile
COPY hello.txt .
```

Copies a file from the Docker build context into the image.

With:

```dockerfile
WORKDIR /app
```

the resulting location is:

```text
/app/hello.txt
```

---

## 23. Dockerfile — `CMD`

```dockerfile
CMD ["cat", "hello.txt"]
```

Defines the default command for the container.

The example container behaved as:

```text
container starts
      ↓
cat hello.txt
      ↓
prints message
      ↓
process exits
      ↓
container exits
```

This demonstrated that a container is associated with its main process. When that process finishes, the container stops.

---

## 24. Building a Custom Image

Executed:

```bash
docker build -t docker-learning:v1 .
```

Breakdown:

```text
docker build
    ↓
Build an image

-t
    ↓
Assign a name/tag

docker-learning:v1
    ↓
Image name and tag

.
    ↓
Current directory as build context
```

The resulting image was checked using:

```bash
docker images
```

---

## 25. Running the Custom Image

Executed:

```bash
docker run --name docker-learning-container docker-learning:v1
```

The container printed:

```text
Hello from my first Docker image!
```

and then exited because its main process (`cat`) finished.

The stopped container was inspected using:

```bash
docker ps -a
```

and its output was checked using:

```bash
docker logs docker-learning-container
```

---

## 26. Docker Image History

Executed:

```bash
docker history docker-learning:v1
```

This connected Dockerfile instructions with image history/layers.

```text
Dockerfile
    ↓
Instructions
    ↓
Image layers/history
```

---

## 27. Build Context

The final `.` in:

```bash
docker build -t docker-learning:v1 .
```

represents the Docker build context.

The current directory was provided as the context from which Docker could access files required during the build.

Example:

```text
docker-learning/
├── Dockerfile
└── hello.txt
```

The Dockerfile could then use:

```dockerfile
COPY hello.txt .
```

because `hello.txt` was inside the build context.

---

## 28. Why Build Context Matters

A real project can contain:

```text
target/
.git/
IDE files
logs/
temporary files
secrets
large files
```

We should not blindly copy all of these into a Docker image.

Unnecessary build-context files can increase build overhead and can expose files to build instructions.

This is why Docker provides:

```text
.dockerignore
```

`.dockerignore` will be covered in a later session.

---

## 29. Today's Key Mental Model

```text
Dockerfile
    │
    │ docker build
    ▼
  Image
    │
    │ docker run
    ▼
Container
    │
    ▼
Main Process
```

And:

```text
Docker CLI
    ↓
Docker Engine
    ↓
Images / Containers / Networks / Volumes
```

---

## 30. KRB Enterprise Connection

We deliberately did not Dockerize KRB Enterprise yet.

The current application contains:

```text
KRB Enterprise
     │
     ├── Spring Boot
     ├── Java
     ├── Spring Security
     ├── JWT
     ├── JPA
     ├── Flyway
     └── PostgreSQL
```

The Docker learning phase is being completed first so that each part of the eventual Docker configuration has a clear purpose.

Eventually:

```text
┌─────────────────────────┐
│ KRB Enterprise          │
│ Spring Boot Container   │
└────────────┬────────────┘
             │
        Docker Network
             │
             ▼
┌─────────────────────────┐
│ PostgreSQL Container    │
└─────────────────────────┘
```

---

## 31. Session Completion

### Completed today

- [x] Why Docker exists
- [x] Docker mental model
- [x] Docker Engine
- [x] Docker CLI
- [x] Images
- [x] Containers
- [x] Docker vs virtual machines
- [x] Image layers
- [x] Base images
- [x] Image tags
- [x] Image digests
- [x] Docker registries
- [x] `docker --version`
- [x] `docker pull`
- [x] `docker images`
- [x] `docker run`
- [x] `docker ps`
- [x] `docker ps -a`
- [x] `docker inspect`
- [x] `docker logs`
- [x] `docker exec`
- [x] `docker stop`
- [x] `docker start`
- [x] `docker rm`
- [x] Dockerfile fundamentals
- [x] `FROM`
- [x] `WORKDIR`
- [x] `COPY`
- [x] `CMD`
- [x] `docker build`
- [x] Build context
- [x] `docker history`
- [x] Custom Docker image creation
- [x] Custom container execution

---

## 32. Explicitly Deferred to Tomorrow

The following topic is **not part of today's notes**:

```text
CMD vs ENTRYPOINT
```

Tomorrow's session will begin with:

```text
CMD vs ENTRYPOINT
        ↓
ENTRYPOINT + CMD
        ↓
Command overriding
        ↓
Spring Boot application process
```

Tomorrow's notes will include that lesson.

---

## KRB Academy Progress

```text
Docker
│
├── Fundamentals                         ✅
├── Architecture                         ✅
├── Images                               ✅
├── Containers                           ✅
├── Image layers                         ✅
├── Registries                           ✅
├── Container lifecycle                  ✅
├── Dockerfile fundamentals              ✅
├── docker build                         ✅
├── Build context                        ✅
├── CMD vs ENTRYPOINT                    ⏭ Tomorrow
├── .dockerignore                        ⏭
├── Docker networking                    ⏭
├── Docker volumes                       ⏭
├── Environment/configuration            ⏭
├── Multi-stage builds                   ⏭
├── Docker Compose                       ⏭
└── KRB Enterprise Dockerization        ⏭
```

---

**End of KRB Academy session — 24 August 2026**
