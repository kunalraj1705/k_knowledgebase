# KRB Academy 

# Docker --- Docker Configuration and Environment Concepts

**Session:** Docker Configuration and Environment Concepts  
**Date:** 30 August 2026  
**Project Context:** KRB Enterprise  
**Status:** Completed

---

## 1. Docker `ENV`
`ENV` defines default environment variables in an image.

```dockerfile
FROM ubuntu:24.04
ENV APP_NAME=krb-enterprise
ENV APP_ENV=development
CMD ["printenv"]
```

`ENV` values are available when the container starts.

## 2. Runtime Override with `-e`
```bash
docker run --rm -e APP_ENV=production environment-demo:v1
```

Runtime configuration overrides the image default.

```text
Dockerfile ENV → default
docker run -e → runtime override
```

## 3. `--env-file`
Example `production.env`:

```text
APP_ENV=production
APP_VERSION=2.0.0
```

```bash
docker run --rm --env-file production.env environment-demo:v1
```

Result:
- `APP_NAME` came from Dockerfile `ENV`
- `APP_ENV` was overridden
- `APP_VERSION` was added at runtime

## 4. Build-Time vs Runtime
```text
docker build → build-time → image creation
docker run   → runtime    → container startup
```

Principle: **Build once, configure at runtime.**

## 5. Docker `ARG`
```dockerfile
ARG APP_VERSION
RUN echo "Building version: $APP_VERSION"
```

Build:

```bash
docker build --build-arg APP_VERSION=1.0.0 -t arg-demo:v1 .
```

`ARG` is available during the build but is not automatically available inside the running container.

## 6. `ARG` → `ENV`
```dockerfile
ARG APP_VERSION
ENV APP_VERSION=$APP_VERSION
```

Build:

```bash
docker build -f Dockerfile.arg-env --build-arg APP_VERSION=2.0.0 -t arg-env-demo:v1 .
```

The running container then contained:

```text
APP_VERSION=2.0.0
```

## 7. `ARG` vs `ENV`

| Feature | `ARG` | `ENV` |
|---|---|---|
| Available during build | Yes | Yes |
| Available at runtime | No, by default | Yes |
| Main purpose | Build-time values | Image/runtime defaults |

## 8. Spring Boot Environment Variables
Example mapping:

```text
spring.datasource.url → SPRING_DATASOURCE_URL
spring.datasource.username → SPRING_DATASOURCE_USERNAME
spring.datasource.password → SPRING_DATASOURCE_PASSWORD
```

Placeholders can also be used:

```properties
spring.datasource.url=${DB_URL}
spring.datasource.username=${DB_USERNAME}
spring.datasource.password=${DB_PASSWORD}
```

## 9. Configuration vs Secrets

Configuration:
- Server port
- Active profile
- Logging level
- Database hostname

Secrets:
- Database password
- Credentials
- JWT signing key
- API keys

## 10. PostgreSQL Initialization Variables
```bash
-e POSTGRES_DB=krb_enterprise
-e POSTGRES_USER=krb
-e POSTGRES_PASSWORD=password
```

These configure the PostgreSQL image during initial database setup. They are not application connection commands.

## 11. Existing Container vs New Container

Existing stopped container:

```bash
docker start krb-postgres
```

New container:

```bash
docker run ...
```

`docker start` starts an existing container; `docker run` creates and starts a new one.

## 12. PostgreSQL Persistence
The Docker volume:

```text
krb_postgres_data
```

stores database data independently from the container lifecycle.

## Today's Main Takeaways
```text
ARG → build-time
ENV → image-level default
-e / --env-file → runtime configuration
Secrets → sensitive deployment values
Spring Profiles → environment-specific configuration
```

## Next Session
Start **Docker Compose**, then apply it to:

```text
KRB Enterprise application
PostgreSQL
Docker network
Persistent volume
Environment-specific configuration
```
