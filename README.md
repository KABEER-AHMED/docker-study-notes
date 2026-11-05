# Docker Study Notes — Midterm Prep

Complete notes covering Docker fundamentals, Dockerfiles, and Docker Compose — written for a DevOps Lab midterm (Docker Swarm intentionally excluded). Includes full command references, flag explanations, and worked exam-style examples.

---

## Table of Contents

1. [Core Concepts](#1-core-concepts)
2. [The Dockerfile](#2-the-dockerfile)
3. [Building & Running — `docker` CLI](#3-building--running--docker-cli)
4. [Networking Basics](#4-networking-basics)
5. [Volumes](#5-volumes)
6. [Docker Compose](#6-docker-compose)
7. [Multi-stage Builds](#7-multi-stage-builds)
8. [`.dockerignore`](#8-dockerignore)
9. [Quick-Reference Cheat Sheet](#9-quick-reference-cheat-sheet)
10. [Common Exam Traps](#10-common-exam-traps)
11. [Worked Example — Task B1: Containerize a Flask App](#11-worked-example--task-b1-containerize-a-flask-app)
12. [Worked Example — Task B2: Reviewing a Bad Dockerfile](#12-worked-example--task-b2-reviewing-a-bad-dockerfile)
13. [Worked Example — Task B3: Multi-stage Node.js Build](#13-worked-example--task-b3-multi-stage-nodejs-build)
14. [Worked Example — Task B4: CMD vs ENTRYPOINT](#14-worked-example--task-b4-cmd-vs-entrypoint)
15. [Worked Example — Task C1: Writing a Compose File](#15-worked-example--task-c1-writing-a-compose-file)
16. [Worked Example — Task C2: Debugging a Compose File](#16-worked-example--task-c2-debugging-a-compose-file)
17. [Worked Example — Task C3: Compose Commands](#17-worked-example--task-c3-compose-commands)

---

## 1. Core Concepts

| Term | Definition |
|---|---|
| **Image** | A read-only template/blueprint (layers of filesystem + metadata) used to create containers. |
| **Container** | A running (or stopped) *instance* of an image — isolated process with its own filesystem, network, etc. |
| **Dockerfile** | A text file with instructions to **build** an image. |
| **Registry** | Where images are stored (Docker Hub is the default public one). |
| **Volume** | Persistent storage that lives outside a container's writable layer. |
| **Docker Compose** | A tool to define and run **multi-container** apps using a YAML file. |

> Analogy: **Image = class, Container = object (instance)**.

---

## 2. The Dockerfile

A Dockerfile is a plain text file (literally named `Dockerfile`, no extension) with instructions, executed top-to-bottom, each creating a new **layer**.

### Core instructions

```dockerfile
# Comment — this is how you comment in a Dockerfile

FROM node:18-alpine
# Base image to build FROM. ALWAYS the first instruction (except ARG before it).
# "alpine" = minimal size variant. "18" = version tag.

LABEL maintainer="kabeer@example.com"
# Metadata, optional.

WORKDIR /app
# Sets the working directory for any RUN/CMD/COPY/ENTRYPOINT after it.
# Creates the dir if it doesn't exist. Avoid using `RUN cd ...` — use this instead.

COPY package.json package-lock.json ./
# Copies files FROM your machine (build context) INTO the image.
# Copying package.json first (before full code) lets Docker cache the npm install layer.

RUN npm install
# Executes a command DURING the build, creates a new layer.
# Common mistake: confusing RUN (build-time) with CMD (runtime).

COPY . .
# Copy everything else (source code) in.

ENV NODE_ENV=production
# Sets an environment variable, available at build time AND when container runs.

ARG BUILD_VERSION=1.0
# Build-time only variable (not available in running container, unless you ENV it too).
# Used with: docker build --build-arg BUILD_VERSION=2.0 .

EXPOSE 3000
# Documentation only! Does NOT actually publish the port.
# You still need -p when running the container.

VOLUME /app/data
# Declares a mount point as a volume (anonymous volume if not bound to a named one).

USER node
# Switch from root to a non-root user (security best practice).

CMD ["node", "server.js"]
# The DEFAULT command when container STARTS. Can be overridden at `docker run` time.
# Exec form (preferred, JSON array) vs shell form: CMD node server.js

ENTRYPOINT ["node"]
# Main command that is NOT easily overridden — args passed to `docker run` get
# APPENDED to it. Often combined with CMD for default args:
# ENTRYPOINT ["node"]
# CMD ["server.js"]   -> runs `node server.js` by default, but `docker run img app.js` -> `node app.js`
```

### Exam-critical distinctions

| | **RUN** | **CMD** | **ENTRYPOINT** |
|---|---|---|---|
| When it runs | **Build time** | **Container start** | **Container start** |
| Purpose | Install stuff, build layers | Default command | Fixed main command |
| Overridable? | N/A | Yes, fully, via `docker run img <cmd>` | Args appended, or override with `--entrypoint` |
| Can have multiple? | Yes (each = new layer) | Only last one wins | Only last one wins |

**Shell form vs Exec form**
- Exec form: `CMD ["executable", "arg1", "arg2"]` — runs directly, no shell, recommended.
- Shell form: `CMD executable arg1 arg2` — runs via `/bin/sh -c`, allows shell features (env var expansion, `&&`) but no proper signal handling (e.g. Ctrl+C / `docker stop` doesn't propagate to the real process, which becomes PID 2 instead of PID 1).

**COPY vs ADD**
- `COPY src dest` — simple file/dir copy. **Prefer this.**
- `ADD src dest` — also can: fetch a URL, and auto-extract local tar archives. Use only when you need that.

### A full example Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

ENV FLASK_APP=app.py
EXPOSE 5000

CMD ["flask", "run", "--host=0.0.0.0"]
```

---

## 3. Building & Running — `docker` CLI

### Build an image
```bash
docker build -t myapp:1.0 .
```
- `-t myapp:1.0` → tag (name:tag). Without a tag, defaults to `latest`.
- `.` → **build context** (the directory sent to the Docker daemon; must contain Dockerfile unless `-f` used).
- `-f path/to/Dockerfile` → use a Dockerfile with a different name/path.
- `--build-arg KEY=value` → pass an `ARG` value.
- `--no-cache` → ignore layer cache, rebuild everything from scratch.
- `--target stageName` → build only up to a specific stage (multi-stage builds).

### Run a container
```bash
docker run -d -p 8080:80 --name webserver -e ENV_VAR=val -v mydata:/app/data myapp:1.0
```
- `-d` → detached (background).
- `-it` → interactive + allocate a TTY (used together for shells: `docker run -it ubuntu bash`).
- `-p HOST:CONTAINER` → publish port. e.g. `-p 8080:80` → host:8080 → container:80.
- `-P` → publish ALL exposed ports to random host ports.
- `--name webserver` → give container a friendly name (else Docker auto-generates one).
- `-e KEY=value` → set environment variable.
- `--env-file .env` → load env vars from a file.
- `-v source:target` → mount a volume (see [Volumes](#5-volumes)).
- `--rm` → auto-remove container when it stops (great for throwaway/test containers).
- `--network mynet` → attach to a specific network.
- `--restart=always` → restart policy (`no`, `on-failure`, `always`, `unless-stopped`).
- `--memory=512m` / `--cpus=1` → resource limits.

### Managing containers
```bash
docker ps              # list running containers
docker ps -a            # list ALL containers (including stopped)
docker stop <id|name>   # graceful stop (SIGTERM, then SIGKILL after timeout)
docker start <id|name>  # start a stopped container
docker restart <id>
docker rm <id>          # remove a stopped container
docker rm -f <id>       # force remove (even if running)
docker logs <id>        # view stdout/stderr
docker logs -f <id>     # follow logs (like tail -f)
docker exec -it <id> bash   # open a shell INSIDE a running container
docker inspect <id>     # full JSON metadata (IP, mounts, env, etc.)
docker stats            # live resource usage
```

### Managing images
```bash
docker images                   # list local images
docker rmi <image>               # remove an image
docker pull nginx:latest         # download image from registry
docker push myrepo/myapp:1.0     # upload image to registry
docker tag myapp:1.0 myrepo/myapp:1.0   # re-tag
docker history myapp:1.0         # see the layers
```

### Cleanup
```bash
docker system prune        # remove unused containers, networks, dangling images
docker system prune -a     # also remove all unused images
docker container prune     # remove all stopped containers
docker volume prune        # remove unused volumes
docker image prune         # remove dangling images
```

---

## 4. Networking Basics

```bash
docker network ls
docker network create mynet
docker network inspect mynet
docker run --network mynet ...
```

Default drivers: **bridge** (default, isolated network on host), **host** (container shares host's network stack), **none** (no networking).

Containers on the **same user-defined bridge network** can reach each other **by container name** (Docker's internal DNS) — this is exactly why Compose services can call each other by service name.

---

## 5. Volumes

Three types of mounts:

| Type | Syntax | Use case |
|---|---|---|
| **Named volume** | `-v myvolume:/path/in/container` | Managed by Docker, persists data, portable, preferred for databases |
| **Bind mount** | `-v /host/path:/path/in/container` | Maps a specific host directory/file — good for dev (live code reload) |
| **Anonymous volume** | `-v /path/in/container` | No name, Docker auto-generates ID, harder to reuse |

```bash
docker volume create mydata       # create a named volume
docker volume ls                  # list volumes
docker volume inspect mydata      # see where it lives on disk
docker volume rm mydata           # delete it (must not be in use)
```

**Why volumes matter:** a container's writable layer is destroyed when the container is removed. Volumes let data (e.g. database files) **survive** container removal/recreation.

**`-v` vs `--mount`** (newer, more explicit syntax):
```bash
docker run --mount type=bind,source=/host/path,target=/container/path myapp
docker run --mount type=volume,source=myvol,target=/data myapp
```
`--mount` is more verbose but clearer and preferred in official docs for precision.

Read-only mount:
```bash
docker run -v mydata:/app/data:ro myapp
```

---

## 6. Docker Compose

`docker-compose.yml` (or `compose.yaml`) defines multi-container apps declaratively.

### Example: a web app + database

```yaml
services:
  web:
    build: .                      # build from local Dockerfile
    # OR: image: myrepo/myapp:1.0  # use a pre-built image instead
    ports:
      - "8080:80"                 # HOST:CONTAINER
    environment:
      - NODE_ENV=production
      - DB_HOST=db                # "db" resolves via Compose's internal DNS!
    volumes:
      - ./src:/app/src            # bind mount (live code)
      - static_data:/app/static   # named volume
    depends_on:
      - db
    networks:
      - appnet
    restart: always

  db:
    image: postgres:15
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: mydb
    volumes:
      - db_data:/var/lib/postgresql/data   # persist DB across restarts
    networks:
      - appnet

volumes:
  static_data:
  db_data:

networks:
  appnet:
    driver: bridge
```

### Key fields

- `build:` → build from a local Dockerfile (can specify `context:` and `dockerfile:` separately).
- `image:` → use an existing image instead of building.
- `ports:` → `"HOST:CONTAINER"`.
- `volumes:` → same idea as `-v`; top-level `volumes:` key declares **named** volumes used across services.
- `environment:` → list or map form, both valid:
  ```yaml
  environment:
    - KEY=value
  # or
  environment:
    KEY: value
  ```
- `env_file: .env` → load variables from a file.
- `depends_on:` → controls **start order** only (NOT "wait until healthy" unless combined with `healthcheck:` + `condition: service_healthy`).
- `networks:` → custom networks; services on the same Compose network can reach each other by **service name** as hostname.
- `command:` → override the default CMD.
- `restart:` → `no` | `always` | `on-failure` | `unless-stopped`.

### Compose CLI commands

```bash
docker compose up              # build (if needed) + start all services, foreground
docker compose up -d           # detached mode
docker compose up --build      # force rebuild images before starting
docker compose down            # stop and remove containers, networks (NOT volumes by default)
docker compose down -v         # also remove named volumes
docker compose ps              # list running compose services
docker compose logs -f         # follow logs of all services
docker compose logs -f web     # logs for just one service
docker compose exec web bash   # shell into a running service
docker compose build           # just build images, don't start
docker compose stop            # stop containers without removing
docker compose restart web     # restart one service
docker compose config          # validate & print the resolved config
```

> Note: older syntax was `docker-compose` (hyphenated, standalone binary). Newer Docker uses `docker compose` (space, built into the CLI as a plugin). Know both — exams sometimes use the old hyphenated form.

---

## 7. Multi-stage Builds

Used to keep final images small (e.g., build with full SDK, ship only the compiled output).

```dockerfile
# Stage 1: build
FROM node:18 AS builder
WORKDIR /app
COPY . .
RUN npm install && npm run build

# Stage 2: final, minimal image
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

`COPY --from=builder` pulls files from an earlier named stage, discarding everything else (build tools, source, node_modules) from the final image.

---

## 8. `.dockerignore`

Like `.gitignore` — excludes files from the **build context** (speeds up builds, avoids leaking secrets/node_modules into the image):

```
node_modules
.git
*.log
.env
Dockerfile
```

---

## 9. Quick-Reference Cheat Sheet

```bash
# Build
docker build -t name:tag .

# Run
docker run -d -p 8080:80 --name c1 -v vol:/data -e KEY=val name:tag

# Inspect/manage
docker ps -a
docker logs -f c1
docker exec -it c1 bash
docker stop c1 && docker rm c1

# Images
docker images
docker rmi name:tag
docker pull/push name:tag

# Volumes
docker volume create v1
docker volume ls / rm / inspect

# Compose
docker compose up -d --build
docker compose down -v
docker compose logs -f
docker compose exec service bash
```

---

## 10. Common Exam Traps

1. `EXPOSE` ≠ publishing a port — you still need `-p`.
2. `CMD` is overridden entirely by `docker run img <cmd>`; `ENTRYPOINT` args get appended, not replaced.
3. Named volume vs bind mount: named volume = Docker-managed path, bind mount = you specify the exact host path.
4. `depends_on` ≠ "wait until ready" — just start order (unless paired with `condition: service_healthy`).
5. Build context — `.` sends **everything** in that folder to the daemon (hence `.dockerignore` matters).
6. Order instructions in a Dockerfile to **maximize cache** (put rarely-changing steps like dependency installs before `COPY . .`).
7. Inside a container, `localhost` refers to the container itself — not another service. Use the **service name** as the hostname instead.
8. Secrets should never be hardcoded with `ENV` in a Dockerfile — inject them at runtime (`-e`, `--env-file`, `.env` + variable substitution, or Docker secrets).

---

## 11. Worked Example — Task B1: Containerize a Flask App

**Requirements:** base image `python:3.12-slim`, workdir `/app`, install deps before copying source (cache optimization), no pip cache, `PYTHONUNBUFFERED=1`, expose 5000, non-root user `app` with UID 10001, exec-form CMD.

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

ENV PYTHONUNBUFFERED=1

EXPOSE 5000

RUN useradd -u 10001 -m app
USER app

CMD ["python", "app.py"]
```

**Why the order matters:** `COPY requirements.txt .` + `RUN pip install` happens *before* `COPY . .`. Docker caches each layer by its instruction + inputs. If you change application code but not `requirements.txt`, the `pip install` layer is reused from cache — much faster rebuilds. If `COPY . .` came first, *any* code change would invalidate the cache for the install step too.

- `--no-cache-dir` → stops pip from keeping a wheel/package cache inside the image layer (smaller image).
- `useradd -u 10001 -m app` creates user `app` with UID 10001 and a home dir; `USER app` switches to it for all subsequent instructions (including the final `CMD`).
- `CMD ["python", "app.py"]` is exec form (JSON array), as required.

**Build and run:**
```bash
docker build -t flaskapp:1.0 .
docker run -d --name flask -p 8080:5000 flaskapp:1.0
```
`-p 8080:5000` → host port 8080 maps to container port 5000 (where Flask listens).

---

## 12. Worked Example — Task B2: Reviewing a Bad Dockerfile

**Original (bad) Dockerfile:**
```dockerfile
FROM python:latest
RUN apt-get update
RUN apt-get install -y curl
COPY . /app
WORKDIR /app
RUN pip install -r requirements.txt
ENV DB_PASSWORD=secret123
CMD python app.py
```

### Problems found

1. **`FROM python:latest`** — unpinned tag. Builds aren't reproducible; "latest" can change underneath you and silently break things. Should pin a specific version, e.g. `python:3.12-slim`.
2. **`COPY . /app` before `WORKDIR /app` and before installing deps** — copies the entire source (including code that changes often) before installing dependencies, so *every* code change invalidates the pip-install cache layer.
3. **Separate `RUN apt-get update` and `RUN apt-get install`** — each `RUN` is its own layer; splitting them means a cached `update` layer can go stale relative to a later `install`. Best practice: combine into one `RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*`.
4. **`ENV DB_PASSWORD=secret123`** — secret baked into an image layer, extractable via `docker history`/layer inspection. Should be injected at runtime instead.
5. **`CMD python app.py`** — shell form. Runs via `/bin/sh -c`, so the process isn't PID 1 directly; signals like `SIGTERM` from `docker stop` aren't forwarded properly, causing slow/forced shutdowns. Should be exec form.
6. **No non-root user** — container runs as root by default, a security risk.
7. **No `--no-cache-dir` on pip / no apt list cleanup** — unnecessarily bloats image size.

### Fixed Dockerfile

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

RUN apt-get update && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*

COPY . .

RUN useradd -u 10001 -m appuser
USER appuser

CMD ["python", "app.py"]
```
(`DB_PASSWORD` removed entirely — pass it at `docker run -e DB_PASSWORD=...` time instead.)

---

## 13. Worked Example — Task B3: Multi-stage Node.js Build

**Scenario:** app is built with `npm run build`, writing to `/app/dist`; server starts with `node dist/server.js` on port 3000.

### Dockerfile

```dockerfile
# ---- Stage 1: build ----
FROM node:20-alpine AS build

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm install

COPY . .
RUN npm run build

# ---- Stage 2: production ----
FROM node:20-alpine

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm install --production

COPY --from=build /app/dist ./dist

USER node
EXPOSE 3000

CMD ["node", "dist/server.js"]
```

Key points:
- `AS build` names the first stage so it can be referenced later.
- `COPY --from=build /app/dist ./dist` pulls only the compiled output from stage 1 — none of stage 1's dev dependencies, source TypeScript, or build tools end up in the final image.
- `npm install --production` (or `npm ci --omit=dev`) in the final stage installs only runtime deps.
- `USER node` — `node:alpine` images ship a built-in non-root `node` user already.
- Dependency manifests copied **before** the rest of the source in both stages (same caching reason as B1).

### `.dockerignore`
```
node_modules
.git
dist
.env
```

### Build only the `build` stage
```bash
docker build --target build -t myapp:build .
```

### Cache behavior after a one-line code edit
- `COPY package.json package-lock.json ./` and `RUN npm install` → **cache reused** (manifests unchanged).
- `COPY . .` → **cache invalidated** (file content changed).
- `RUN npm run build` → **reruns** (any layer after an invalidated one reruns too).

---

## 14. Worked Example — Task B4: CMD vs ENTRYPOINT

Dockerfile:
```dockerfile
ENTRYPOINT ["python", "app.py"]
CMD ["--port", "8080"]
```

**a)** `docker run tool:1.0` →
```
python app.py --port 8080
```
`CMD` supplies default arguments appended to `ENTRYPOINT`.

**b)** `docker run tool:1.0 --port 9090` →
```
python app.py --port 9090
```
Anything passed after the image name on `docker run` **replaces `CMD` entirely**, and gets appended to `ENTRYPOINT`.

**c)** Image `web:1.0` has only `CMD ["npm", "start"]` (no `ENTRYPOINT`). `docker run web:1.0 npm test` →
```
npm test
```
With no `ENTRYPOINT`, anything after the image name **replaces `CMD` entirely** — the whole command becomes `npm test`.

**d)** Why exec form over shell form: exec form (`["executable", "arg1"]`) runs the process directly as PID 1, without going through `/bin/sh -c`. This means signals (`SIGTERM` from `docker stop`) go straight to the app, enabling graceful shutdown. Shell form wraps the command in a shell, which becomes PID 1 instead and may not forward signals to the actual application process, causing Docker to wait out the full timeout and then `SIGKILL` it.

---

## 15. Worked Example — Task C1: Writing a Compose File

**Scenario:** `web` service built from `./web` (listens on 3000), published on host 8080, needs `DB_HOST`/`REDIS_HOST` pointing at the other services by name, must wait for a healthy database and a started cache, restarts unless stopped manually. `db` is `postgres:16-alpine`, password from `.env`, must refuse to start without it, data in a named volume, with a healthcheck. `cache` is `redis:7-alpine`. Only `web` publishes a port.

```yaml
services:
  web:
    build: ./web
    ports:
      - "8080:3000"
    environment:
      DB_HOST: db
      REDIS_HOST: cache
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_started
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD:?DB_PASSWORD is required}
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "postgres"]
      interval: 5s
      timeout: 3s
      retries: 5

  cache:
    image: redis:7-alpine

volumes:
  pgdata:
```

`.env`:
```
DB_PASSWORD=changeme123
```

Notes:
- `DB_HOST: db` / `REDIS_HOST: cache` — Compose's internal DNS resolves other services by their **service name** on the default network.
- `depends_on` with `condition: service_healthy` requires `db`'s healthcheck to pass before `web` starts; `condition: service_started` just waits for the container to start (no healthcheck defined for `cache`).
- `restart: unless-stopped` → restarts automatically on crash/reboot, but not if manually stopped.
- `${DB_PASSWORD:?DB_PASSWORD is required}` — Compose errors out if the variable is unset/empty.
- No `ports:` on `db` or `cache` — only `web` publishes a port; internal services stay reachable over the Compose network without exposing to the host.

---

## 16. Worked Example — Task C2: Debugging a Compose File

**Original (buggy) file:**
```yaml
version: "3.8"
services:
  api:
    build: ./api
    ports:
      - 3000:3000
    environment:
      DB_HOST: localhost
      DB_PASSWORD: mysecret
    depends_on:
      - db
  db:
    image: postgres:latest
    ports:
      - "5432:5432"
    environment:
      POSTGRES_PASSWORD: mysecret
    volumes:
      - pgdata:/var/lib/postgresql/data
```

### Problems found

1. **`DB_HOST: localhost`** — wrong hostname. Inside `api`'s own container, `localhost` refers to itself, not the `db` container. Must be `db` (the service name).
2. **`image: postgres:latest`** — unpinned tag; not reproducible. Pin a version like `postgres:16-alpine`.
3. **Hardcoded passwords in the file** — `POSTGRES_PASSWORD`/`DB_PASSWORD` committed in plaintext. Should come from `.env` / `${VAR}` substitution instead.
4. **`db` publishes `5432:5432` to the host** — unnecessary exposure; `api` can reach `db` over the internal Compose network without a host port mapping. Should be removed.
5. **No healthcheck on `db`, combined with plain `depends_on: - db`** — only waits for the container to *start*, not for Postgres to be ready to accept connections. `api` can crash-loop trying to connect too early. Needs a `healthcheck` + `condition: service_healthy`.
6. **No top-level `volumes:` declaration** for `pgdata` — used under `db.volumes` but never declared at the top level; best practice (and sometimes required) to declare it explicitly.
7. *(bonus)* `version: "3.8"` is deprecated/ignored by modern Compose.

### Corrected file

```yaml
services:
  api:
    build: ./api
    ports:
      - "3000:3000"
    environment:
      DB_HOST: db
      DB_PASSWORD: ${DB_PASSWORD:?DB_PASSWORD is required}
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD:?DB_PASSWORD is required}
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "postgres"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  pgdata:
```

---

## 17. Worked Example — Task C3: Compose Commands

All run from the folder containing `compose.yaml`.

```bash
# a) Rebuild images and start everything in background
docker compose up -d --build
```
```bash
# b) Show state of services, including health
docker compose ps
```
```bash
# c) Follow last 50 log lines of web
docker compose logs -f --tail=50 web
```
```bash
# d) Shell inside the running web container
docker compose exec web sh
```
```bash
# e) Run npm test once in a new disposable web container
docker compose run --rm web npm test
```
```bash
# f) Stop and remove containers/networks, keep volumes (DB data)
docker compose down
```
```bash
# g) Same as f), but also delete named volumes
docker compose down -v
```
```bash
# h) Start the stack with 3 replicas of a stateless service "worker"
docker compose up -d --scale worker=3
```

Clarifications:
- `docker compose exec` runs in an **existing, running** container; `docker compose run` **creates a new one** (useful for one-off commands like tests) — `--rm` cleans it up after exit.
- `down` (no `-v`) leaves named volumes intact — Compose separates container/network teardown from volume deletion so you don't lose data by accident.
- `--scale service=N` requires that service to **not** define a fixed container `name:` or a host port that would conflict across replicas.

---

*Notes compiled for DevOps Lab midterm prep — Docker fundamentals, Dockerfile, and Docker Compose only (Swarm and MCQs excluded by request).*
