# Docker

Reach for this when writing a Dockerfile, wiring up Compose, or debugging a container.

## Multi-stage, non-root Dockerfile (the only way you ship)

```dockerfile
# Build stage
FROM eclipse-temurin:25-jdk-alpine AS builder
WORKDIR /app
COPY .mvn .mvn
COPY mvnw pom.xml ./
RUN ./mvnw dependency:go-offline        # cached unless pom.xml changes
COPY src ./src
RUN ./mvnw package -DskipTests

# Runtime stage
FROM eclipse-temurin:25-jre-alpine
RUN addgroup -S app && adduser -S app -G app
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
USER app                                 # never run as root in prod
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Rules: pin the tag (never `:latest`), non-root user, no secrets baked in, JRE (not JDK) at runtime.

## Layer caching - order matters

Put the **least-frequently-changed** things first. Copy dependency manifests and
resolve deps *before* copying source, so a code change doesn't re-download the world.
Each instruction is a cached layer; the first changed layer busts everything after it.

## .dockerignore

```
target/
.git/
.idea/
*.md
.env
```

## CLI - daily

```bash
docker build -t myapp:1.0 .
docker run -d --name myapp -p 8080:8080 --env-file .env myapp:1.0
docker ps                       # running;  docker ps -a  = all
docker logs -f myapp            # follow logs
docker exec -it myapp sh        # shell inside (bash if present)
docker stop myapp && docker rm myapp
docker stats                    # live CPU/mem per container
docker inspect myapp            # full JSON (IP, mounts, env)
```

## CLI - images & cleanup

```bash
docker images
docker rmi myapp:1.0
docker pull postgres:17-alpine
docker tag myapp:1.0 registry.example.com/myapp:1.0
docker push registry.example.com/myapp:1.0

docker system df                # disk usage
docker system prune -a          # nuke unused images/containers/networks (careful)
docker builder prune            # clear build cache
```

## Compose

```yaml
services:
  app:
    build: .
    ports: ["8080:8080"]
    env_file: .env
    depends_on:
      db:
        condition: service_healthy      # wait for real readiness, not bare depends_on
  db:
    image: postgres:17-alpine           # pinned tag, never latest
    environment:
      POSTGRES_DB: app
      POSTGRES_PASSWORD: secret
    volumes:
      - pgdata:/var/lib/postgresql/data # named volume, not anonymous
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  pgdata:
```

```bash
docker compose up -d
docker compose logs -f app
docker compose ps
docker compose down            # add -v to also remove named volumes
docker compose up -d --build   # rebuild changed images
```

## Debugging a broken container

```bash
docker logs myapp                       # what did it print before dying?
docker inspect myapp | grep -A5 Health  # healthcheck status
docker run -it --entrypoint sh myapp:1.0   # bypass entrypoint, poke around
docker exec -it myapp env               # what env does it actually see?
docker port myapp                       # what's actually published?
```

## Gotchas / things I always forget

- **`EXPOSE` documents, it doesn't publish.** You still need `-p 8080:8080`.
- `-p host:container` - host port first. Mixing these up is the #1 "why can't I reach it".
- `depends_on` alone waits for **start**, not **readiness**. Use `condition: service_healthy`.
- Anonymous volumes pile up silently. Name them, and `down -v` to actually clear data.
- Alpine uses `sh`, not `bash`, and musl not glibc - some binaries misbehave. Use a `-jre` base for Java.
- Each `RUN` is a layer; chain with `&&` and clean in the same layer or the cruft persists in a lower layer.
- `docker system prune -a` deletes **all** images not tied to a running container - including ones you wanted.
- Build context is the whole dir sent to the daemon. A missing `.dockerignore` sends `target/`, `.git/`, etc. → slow builds.
- Env vars in `ENV` are baked into the image (visible in history). Secrets go via `--env-file`/runtime, never `ENV`.

## Quick reference

| Task | Command |
|---|---|
| Shell in | `docker exec -it NAME sh` |
| Follow logs | `docker logs -f NAME` |
| Publish port | `-p HOST:CONTAINER` |
| Free disk | `docker system prune -a` |
| Wait for dep | `depends_on: { db: { condition: service_healthy } }` |
| Rebuild | `docker compose up -d --build` |
| Remove w/ volumes | `docker compose down -v` |
