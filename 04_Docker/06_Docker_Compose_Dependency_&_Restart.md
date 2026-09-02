# KRB Academy --- Docker

## Docker Compose: Service Dependencies, Health Checks & Restart Policies

**Session:** Docker Compose\
**Date:** 2 September 2026\
**Project Context:** KRB Enterprise\
**Status:** Completed

------------------------------------------------------------------------

# 1. Session Objective

Today's session continued Docker Compose and focused on:

-   `depends_on`
-   Service startup ordering
-   Health checks
-   `condition: service_healthy`
-   Container health states
-   Restart policies
-   Practical testing of restart behavior

The main objective was to understand the difference between:

``` text
Container started
        ≠
Application ready
```

and:

``` text
Startup orchestration
        ≠
Runtime recovery
```

------------------------------------------------------------------------

# 2. Container Started vs Application Ready

A container starting successfully does **not** necessarily mean the
application inside it is ready.

Mental model:

``` text
Container starts
      ↓
Application initializes
      ↓
Application may still be starting
      ↓
Application becomes ready
```

This is why health checks are important.

------------------------------------------------------------------------

# 3. `depends_on` --- Short Syntax

Configuration:

``` yaml
depends_on:
  - postgres
```

Meaning:

``` text
PostgreSQL starts first
        ↓
Dependent service starts
```

This controls **startup ordering**.

It does **not** wait for PostgreSQL to become healthy.

------------------------------------------------------------------------

# 4. `depends_on` --- Long Syntax

Configuration:

``` yaml
depends_on:
  postgres:
    condition: service_healthy
```

Meaning:

``` text
PostgreSQL starts
        ↓
Health checks run
        ↓
PostgreSQL becomes healthy
        ↓
Dependent service starts
```

This is different from short syntax because the dependent service waits
for the dependency to become healthy.

------------------------------------------------------------------------

# 5. PostgreSQL Health Check

Health check configuration:

``` yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U krb -d krb_enterprise"]
  interval: 10s
  timeout: 5s
  retries: 5
  start_period: 10s
```

## Understanding each property

### `test`

The command Docker executes to determine container health.

### `interval`

How often Docker runs the health check.

``` text
Every 10 seconds
```

### `timeout`

Maximum time allowed for one health-check execution.

### `retries`

Number of consecutive failures before Docker marks the container as:

``` text
unhealthy
```

### `start_period`

Initial grace period while the service is starting.

------------------------------------------------------------------------

# 6. Running vs Healthy

Important concept:

``` text
Running ≠ Healthy
```

A container can be:

``` text
Running
   +
Unhealthy
```

Practical flow:

``` text
Container process running
        ↓
Health check fails
        ↓
Container remains running
        ↓
Health status becomes unhealthy
```

Health status and container lifecycle are separate concepts.

------------------------------------------------------------------------

# 7. Intentional Unhealthy Test

The health check was temporarily changed to:

``` yaml
test: ["CMD-SHELL", "exit 1"]
```

`exit 1` causes the health-check command to fail.

Flow:

``` text
PostgreSQL starts
        ↓
Health check returns exit code 1
        ↓
Health checks continue failing
        ↓
Retry limit reached
        ↓
Container becomes unhealthy
```

The PostgreSQL container remained running, but its health status became
unhealthy.

------------------------------------------------------------------------

# 8. `condition: service_healthy` Practical Result

With:

``` yaml
depends_on:
  postgres:
    condition: service_healthy
```

Compose waited for PostgreSQL.

When the health check intentionally failed, Compose reported that the
dependency was unhealthy and the dependent service did not start.

Mental model:

``` text
Dependency unhealthy
        ↓
service_healthy condition not satisfied
        ↓
Dependent service does not start
```

------------------------------------------------------------------------

# 9. Inspecting Health Status

Command used:

``` bash
docker inspect --format='{{json .State.Health}}' compose-demo-postgres-1
```

Successful health checks returned:

``` text
/var/run/postgresql:5432 - accepting connections
```

Container status later showed:

``` text
Up ... (healthy)
```

------------------------------------------------------------------------

# 10. `depends_on` Is Not Runtime Recovery

`depends_on` mainly controls startup dependency behavior.

It does not mean:

``` text
PostgreSQL crashes
        ↓
Ubuntu automatically stops
```

It also does not automatically provide complete runtime recovery for
dependent applications.

Important distinction:

``` text
Startup orchestration
        ≠
Runtime recovery
```

------------------------------------------------------------------------

# 11. Restart Policies

Restart policies define what Docker should do when the container's main
process stops.

Policies tested:

``` text
on-failure
unless-stopped
always
```

------------------------------------------------------------------------

# 12. `restart: on-failure`

Test configuration:

``` yaml
restart-test:
  image: ubuntu:24.04
  command: bash -c "echo Container started; sleep 5; exit 1"
  restart: on-failure
```

Flow:

``` text
Container starts
        ↓
sleep 5
        ↓
exit 1
        ↓
Process fails
        ↓
Docker restarts container
```

Inspection showed `RestartCount` increasing.

When the command was changed to:

``` bash
exit 0
```

the container exited normally and was not restarted.

Rule:

``` text
Exit 0        → No restart
Non-zero exit → Restart
```

------------------------------------------------------------------------

# 13. `restart: unless-stopped`

Behavior observed:

``` text
Container exits normally
        ↓
Docker restarts it
```

and:

``` text
Container fails
        ↓
Docker restarts it
```

After explicitly stopping the container:

``` bash
docker stop <container-name>
```

the container remained stopped.

Mental model:

``` text
Keep restarting
unless explicitly stopped
```

------------------------------------------------------------------------

# 14. `restart: always`

With:

``` yaml
restart: always
```

the test process exited with:

``` bash
exit 0
```

Docker still restarted the container.

Observed state:

``` text
Restarting (0)
```

The `(0)` indicated successful process exit, while the restart policy
caused Docker to start the container again.

------------------------------------------------------------------------

# 15. Restart Policy Comparison

  Restart Policy     Exit 0       Non-zero Exit   Explicit `docker stop`
  ------------------ ------------ --------------- ------------------------
  `on-failure`       No restart   Restart         Stopped
  `unless-stopped`   Restart      Restart         Remains stopped
  `always`           Restart      Restart         Stopped explicitly

------------------------------------------------------------------------

# 16. Commands Practiced

``` bash
docker-compose up
```

``` bash
docker-compose down
```

``` bash
docker ps
```

``` bash
docker ps -a
```

``` bash
docker kill compose-demo-postgres-1
```

``` bash
docker stop <container-name>
```

``` bash
docker start <container-name>
```

``` bash
docker inspect <container-name>
```

Inspect health:

``` bash
docker inspect --format='{{json .State.Health}}' <container-name>
```

Inspect restart policy:

``` bash
docker inspect --format='{{json .HostConfig.RestartPolicy}}' <container-name>
```

Inspect status and restart count:

``` bash
docker inspect --format='Status: {{.State.Status}} | RestartCount: {{.RestartCount}}' <container-name>
```

------------------------------------------------------------------------

# 17. KRB Enterprise Connection

Conceptual architecture:

``` text
KRB Enterprise Spring Boot
            │
            │ connects using
            ▼
       postgres:5432
            │
            ▼
       PostgreSQL
            │
            ▼
      Persistent Volume
```

Docker Compose health checks can help coordinate startup.

However, the application should also handle temporary database
unavailability during runtime.

------------------------------------------------------------------------

# 18. Key Takeaways

1.  Container started does not mean application ready.
2.  Running does not mean healthy.
3.  Health checks and restart policies are separate mechanisms.
4.  Short `depends_on` controls startup ordering.
5.  `condition: service_healthy` waits for dependency readiness.
6.  An unhealthy dependency can prevent dependent service startup.
7.  `depends_on` does not provide complete runtime recovery.
8.  `on-failure` restarts after a failed process exit.
9.  `unless-stopped` restarts unless explicitly stopped.
10. `always` restarts after both successful and failed exits.

------------------------------------------------------------------------

# 19. Session Completion

-   [x] `depends_on` short syntax
-   [x] `depends_on` long syntax
-   [x] `condition: service_healthy`
-   [x] Container started vs application ready
-   [x] PostgreSQL health checks
-   [x] Healthy vs unhealthy states
-   [x] Intentional health-check failure
-   [x] Health inspection
-   [x] Startup orchestration vs runtime recovery
-   [x] `restart: on-failure`
-   [x] `restart: unless-stopped`
-   [x] `restart: always`
-   [x] Restart count inspection
-   [x] Practical restart-policy testing

------------------------------------------------------------------------

# 20. Next Session

## Runtime Dependency Recovery

Next practical test:

``` text
PostgreSQL crashes
        ↓
Restart policy restarts PostgreSQL
        ↓
Observe dependent container behavior
        ↓
Understand application-level reconnection
```

Topics to continue:

-   Runtime dependency recovery
-   Application-to-database reconnection
-   Startup orchestration vs runtime resilience

------------------------------------------------------------------------

# KRB Academy Progress

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
├── depends_on                          Completed
├── Health Checks                       Completed
├── Restart Policies                    Completed
├── Runtime Dependency Recovery         Next
├── Compose Environment/Secrets         Upcoming
├── Multi-stage Builds                  Upcoming
└── KRB Enterprise Dockerization        Upcoming
```

------------------------------------------------------------------------

**End of KRB Academy Docker Session --- 2 September 2026**
