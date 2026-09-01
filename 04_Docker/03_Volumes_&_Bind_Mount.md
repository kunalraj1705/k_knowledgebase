# KRB Academy

# Docker — Volumes & Persistent Storage

**Session:** Volumes & Persistent Storage  
**Date:** 27 August 2026  
**Project Context:** KRB Enterprise  
**Status:** Completed

---

## 1. Session Objective

Today's session focused on Docker storage and persistence:

- Container filesystem
- Container stop vs removal
- Docker volumes
- Volume persistence
- PostgreSQL persistent storage
- Bind mounts
- Volume vs bind mount
- Read-only mounts

---

## 2. Container Filesystem

Every container has its own writable filesystem.

```text
Container
│
├── Application files
├── System files
└── Writable container layer
```

Data stored only in the container's writable layer belongs to that container.

---

## 3. Container Stop vs Container Removal

### Container stops

Stopping a container does not delete it.

```text
Container
    ↓
Writes data
    ↓
docker stop
    ↓
Data remains
```

The same container can later be started again:

```bash
docker start container-name
```

### Container is removed

```text
Container
    ↓
Writes data
    ↓
docker rm
    ↓
Container removed
    ↓
Container-only data removed
```

This is why persistent services such as databases should not rely only on the container writable layer.

---

## 4. Docker Volumes

A Docker volume provides storage independent of a specific container.

```text
PostgreSQL Container
        │
        ▼
Docker Volume
        │
        ▼
Persistent Data
```

Key principle:

```text
Container = replaceable runtime
Volume    = persistent storage
```

A container can be removed while the named volume remains.

---

## 5. Creating and Inspecting a Volume

Create:

```bash
docker volume create docker-data
```

List volumes:

```bash
docker volume ls
```

Inspect:

```bash
docker volume inspect docker-data
```

---

## 6. Mounting a Volume

Example:

```bash
docker run -dit   --name volume-demo   -v docker-data:/data   ubuntu:24.04
```

Meaning:

```text
docker-data
     │
     │ mounted into
     ▼
Container /data
```

---

## 7. Volume Persistence Exercise

Data was written into the mounted volume:

```bash
docker exec volume-demo sh -c 'echo "KRB Academy" > /data/test.txt'
```

The original container could be stopped and removed:

```bash
docker stop volume-demo
docker rm volume-demo
```

The volume remained independent of the deleted container.

A new container can mount the same volume and access the existing data.

---

## 8. Existing KRB PostgreSQL Volume

The existing Docker volume on the system was:

```text
krb_postgres_data
```

After deleting the learning volume `docker-data`, this volume remained.

This demonstrated that each Docker volume is an independent resource.

Inspection showed:

```json
[
  {
    "Driver": "local",
    "Mountpoint": "/var/lib/docker/volumes/krb_postgres_data/_data",
    "Name": "krb_postgres_data",
    "Scope": "local"
  }
]
```

### Important fields

**Name**

```text
krb_postgres_data
```

The Docker volume name.

**Driver**

```text
local
```

The volume uses local storage managed by Docker.

**Mountpoint**

```text
/var/lib/docker/volumes/krb_postgres_data/_data
```

The host-side location where Docker manages the volume data.

Do not manually modify PostgreSQL files there while PostgreSQL is using the volume.

**Scope**

```text
local
```

The volume belongs to the current Docker host.

---

## 9. PostgreSQL Persistence

Without persistent storage:

```text
PostgreSQL Container
        ↓
Container filesystem
        ↓
Container removed
        ↓
Database data can be lost
```

With a volume:

```text
PostgreSQL Container
        ↓
PostgreSQL data directory
        ↓
krb_postgres_data
        ↓
Persistent database data
```

Conceptually:

```text
Old PostgreSQL container
        ↓
Removed
        ↓
Volume remains
        ↓
New PostgreSQL container
        ↓
Same volume
        ↓
Database data remains
```

---

## 10. Volume Is Not a Backup

A volume provides persistence across normal container replacement.

It is not automatically protection against:

- Accidental volume deletion
- Database corruption
- Application errors
- Host or storage failure

Therefore:

```text
Persistent storage ≠ Backup strategy
```

---

# 11. Bind Mounts

A bind mount connects a specific host path to a path inside a container.

```text
Ubuntu Host
      │
      ▼
Specific directory
      │
      ▼
Container directory
```

Example:

```bash
-v /host/path:/container/path
```

Unlike a named volume, the developer explicitly chooses the host directory.

---

## 12. Bind Mount Exercise

A host directory was created:

```bash
mkdir -p ~/docker-learning/bind-mount-demo/data
```

A file was created on the host:

```bash
echo "Hello from Ubuntu host" > ~/docker-learning/bind-mount-demo/data/message.txt
```

The directory was mounted into a container:

```bash
docker run -dit   --name bind-mount-demo   -v ~/docker-learning/bind-mount-demo/data:/data   ubuntu:24.04
```

### Host → Container verification

Inside the container:

```bash
ls -l /data
cat /data/message.txt
```

The file created on the host was visible inside the container.

### Container → Host verification

Inside the container:

```bash
echo "Hello from Docker container" > /data/container-message.txt
```

On the host:

```bash
cat ~/docker-learning/bind-mount-demo/data/container-message.txt
```

The file created inside the container was visible on the host.

This proved:

```text
Host files
    ↕
Bind Mount
    ↕
Container files
```

Both sides access the same underlying directory.

---

## 13. Docker Volume vs Bind Mount

### Docker Volume

```bash
-v docker-data:/data
```

```text
Docker
   ↓
Manages storage
   ↓
Named Volume
```

Best suited for:

- PostgreSQL data
- Persistent service data
- Container-managed state

### Bind Mount

```bash
-v /host/path:/data
```

```text
Developer chooses host directory
        ↓
Mounted into container
```

Useful for:

- Local development
- Source files
- Configuration files
- Direct host-file access

---

## 14. Read-only Bind Mount

A bind mount can be mounted read-only:

```bash
-v /host/path:/data:ro
```

Conceptually:

```text
Host
 │
 │ Read-only
 ▼
Container
```

Useful for:

- Configuration files
- Certificates
- Static reference data

---

## 15. Security Consideration

A writable bind mount allows the container to modify the mounted host directory.

```text
Container
    ↓
Write / Delete
    ↓
Host filesystem changes
```

Only mount directories that the container actually needs.

Use read-only mounts where write access is unnecessary.

---

## 16. Three Storage Concepts

### Container Writable Layer

```text
Lives with container
        ↓
Container removed
        ↓
Container-only data removed
```

### Docker Volume

```text
Managed by Docker
        ↓
Independent of a specific container
        ↓
Container removed
        ↓
Volume remains
```

### Bind Mount

```text
Specific host directory
        ↓
Mounted into container
        ↓
Host and container access same files
```

---

## 17. KRB Enterprise Connection

The future architecture will conceptually be:

```text
                 Docker Network
                       │
          ┌────────────┴────────────┐
          │                         │
          ▼                         ▼
┌──────────────────┐      ┌──────────────────┐
│ KRB Enterprise   │─────▶│ PostgreSQL       │
│ Spring Boot      │      │ Container        │
│ Container        │      └────────┬─────────┘
└──────────────────┘               │
                                   ▼
                            Docker Volume
                                   │
                                   ▼
                           Database Data
```

The intended separation is:

```text
Spring Boot
    ↓
Replaceable container runtime

PostgreSQL
    ↓
Containerized database process
    ↓
Persistent state stored in a Docker volume
```

---

## 18. Key Takeaways

1. Stopping a container does not remove it.
2. Removing a container removes data stored only in its writable layer.
3. Docker volumes provide storage independent of a specific container.
4. A named volume can be reused by replacement containers.
5. `krb_postgres_data` is an existing Docker-managed local volume.
6. Bind mounts connect a specific host path to a container path.
7. Writable bind mounts allow two-way file changes.
8. Docker volumes are suitable for persistent database storage.
9. Bind mounts are especially useful during development.
10. Persistent storage is not a replacement for backups.

---

## 19. Session Completion

- [x] Container filesystem
- [x] Container stop vs removal
- [x] Docker volumes
- [x] Volume lifecycle
- [x] Volume inspection
- [x] Volume persistence
- [x] Existing PostgreSQL volume inspection
- [x] PostgreSQL persistence concept
- [x] Volume vs backup
- [x] Bind mounts
- [x] Host-to-container file access
- [x] Container-to-host file access
- [x] Volume vs bind mount
- [x] Read-only bind mounts
- [x] Storage security considerations

---

## 20. Next Session

# Docker Environment Variables & Configuration

We will start with:

```text
ENV
-e
--env-file
```

Then learn:

```text
Build-time configuration
        vs
Runtime configuration
```

Finally, we will connect Docker configuration directly to KRB Enterprise and Spring Boot configuration.

We will cover configuration such as:

```text
Database URL
Database username
Database password
Spring profile
JWT configuration
```

The goal is to understand how configuration can be supplied without unnecessarily hard-coding environment-specific values into a Docker image.

---

## KRB Academy Progress

```text
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
├── Environment/Configuration           Next
├── Multi-stage Builds                  Upcoming
├── Docker Compose                      Upcoming
└── KRB Enterprise Dockerization        Upcoming
```

---

**End of KRB Academy session — 27 August 2026**
