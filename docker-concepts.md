# Docker — A Beginner-Friendly Comprehensive Guide

> Everything you need to understand Docker, from the very first "what is a container" all the way to multi-stage builds, networking, volumes, Docker Compose, and production best practices. Every concept paired with simple analogies and runnable examples so you build a real mental model — not just memorize commands.

---

## Table of Contents

1. [The Problem Docker Solves](#1-the-problem-docker-solves)
2. [What Is Docker? (Plain English)](#2-what-is-docker-plain-english)
3. [Containers vs Virtual Machines](#3-containers-vs-virtual-machines)
4. [Images, Containers & Registries — The Three Big Words](#4-images-containers--registries--the-three-big-words)
5. [Installing Docker & Your First Container](#5-installing-docker--your-first-container)
6. [The Docker Architecture](#6-the-docker-architecture)
7. [Understanding the Dockerfile](#7-understanding-the-dockerfile)
8. [Building Your First Image](#8-building-your-first-image)
9. [Image Layers & Caching](#9-image-layers--caching)
10. [Essential Docker Commands](#10-essential-docker-commands)
11. [Container Lifecycle](#11-container-lifecycle)
12. [Port Mapping — Exposing Containers to the World](#12-port-mapping--exposing-containers-to-the-world)
13. [Volumes & Persistent Data](#13-volumes--persistent-data)
14. [Environment Variables & Configuration](#14-environment-variables--configuration)
15. [Docker Networking](#15-docker-networking)
16. [Docker Compose — Multi-Container Apps Made Easy](#16-docker-compose--multi-container-apps-made-easy)
17. [Multi-Stage Builds](#17-multi-stage-builds)
18. [Image Tags & Versioning](#18-image-tags--versioning)
19. [Docker Hub & Other Registries](#19-docker-hub--other-registries)
20. [`.dockerignore` & Build Context](#20-dockerignore--build-context)
21. [Logs, Exec, Inspect — Debugging Containers](#21-logs-exec-inspect--debugging-containers)
22. [Security Basics](#22-security-basics)
23. [Production Best Practices](#23-production-best-practices)
24. [Common Pitfalls & Troubleshooting](#24-common-pitfalls--troubleshooting)
25. [Practical Use Cases](#25-practical-use-cases)
26. [Docker vs Kubernetes — Where Each Fits](#26-docker-vs-kubernetes--where-each-fits)
27. [Interview Q&A](#27-interview-qa)
28. [Quick Command Cheat Sheet](#28-quick-command-cheat-sheet)

---

## 1. The Problem Docker Solves

Before Docker, deploying software was painful. The classic phrase was:

> *"But it works on my machine!"*

### Why apps broke when moved

An app on your laptop depends on hundreds of things:
- **OS** — Ubuntu? macOS? Windows?
- **Language version** — Node 18 or Node 20? Python 3.10 or 3.12?
- **Libraries** — installed globally? specific versions?
- **System tools** — `imagemagick`, `ffmpeg`, `openssl`...
- **Environment variables** — database URLs, API keys.
- **File paths** — `/home/abhi/...` vs `/Users/abhi/...`.

Move the app to another machine, miss one of these, and it breaks.

### The shipping container analogy

Before the 1950s, cargo shipping was chaos — every product (sacks, barrels, boxes) had a different shape, so loading and unloading ships took weeks. Then **Malcolm McLean** invented the standardized **shipping container**: one universal box, any size port, any ship, any truck.

**Docker did the same for software.** Package your app + its dependencies + its config into a standardized "container", and it runs the same way on any machine that has Docker — your laptop, a coworker's Mac, a Linux server, AWS, Azure, anywhere.

> **"Build once, run anywhere."**

---

## 2. What Is Docker? (Plain English)

**Docker** is a tool that lets you package an application together with everything it needs — code, runtime, system libraries, config — into a single bundle called a **container**, which can run on any machine identically.

### Plain analogies

- **Container = lunchbox.** Inside: the food (your app), the fork, the napkin, the salt packet (everything it needs). You can carry it anywhere and eat the same lunch.
- **Image = recipe.** The blueprint for making a container. Write it once, anyone can cook the same dish.
- **Docker Engine = the kitchen.** The thing that reads the recipe and produces the lunchbox.

### Why Docker became huge

1. **Consistency** — dev, staging, prod all run the same image.
2. **Isolation** — each container has its own filesystem and processes; apps don't conflict.
3. **Lightweight** — containers share the host's OS kernel, so they boot in seconds and use a fraction of a VM's RAM.
4. **Portability** — any machine with Docker can run any Docker image.
5. **Fast deploys** — pull an image, start the container, done.

---

## 3. Containers vs Virtual Machines

This is the #1 question beginners ask. Let's settle it visually.

### Virtual Machines

```
┌─────────────────────────────────────────────────────────┐
│   VM 1            VM 2            VM 3                  │
│  ┌──────┐        ┌──────┐        ┌──────┐               │
│  │ App  │        │ App  │        │ App  │               │
│  ├──────┤        ├──────┤        ├──────┤               │
│  │ Libs │        │ Libs │        │ Libs │               │
│  ├──────┤        ├──────┤        ├──────┤               │
│  │ Guest│        │ Guest│        │ Guest│   ← Full OS   │
│  │  OS  │        │  OS  │        │  OS  │     per VM!   │
│  └──────┘        └──────┘        └──────┘               │
│  ─────────── Hypervisor (VMware, VirtualBox) ─────────  │
│  ─────────────── Host Operating System ───────────────  │
│  ─────────────────── Physical Hardware ───────────────  │
└─────────────────────────────────────────────────────────┘
```

Each VM has a **complete operating system** — many gigabytes, slow boot, heavy.

### Containers

```
┌─────────────────────────────────────────────────────────┐
│   Container 1    Container 2    Container 3            │
│  ┌──────┐        ┌──────┐        ┌──────┐               │
│  │ App  │        │ App  │        │ App  │               │
│  ├──────┤        ├──────┤        ├──────┤               │
│  │ Libs │        │ Libs │        │ Libs │ ← No OS!     │
│  └──────┘        └──────┘        └──────┘               │
│  ────────────── Docker Engine ───────────────────────  │
│  ─────── Host Operating System (shared kernel) ─────  │
│  ─────────────────── Physical Hardware ───────────────  │
└─────────────────────────────────────────────────────────┘
```

Containers share the host's OS kernel. No guest OS — just the app and its libraries. **Tiny and fast.**

### Side-by-side comparison

| | Virtual Machine | Container |
|---|---|---|
| **Size** | GBs (full OS image) | MBs (app + libs only) |
| **Boot time** | 30s – 2 min | < 1 second |
| **RAM overhead** | Hundreds of MB per VM | A few MB per container |
| **Isolation** | Strong (separate OS) | Weaker (shared kernel) |
| **Density per host** | ~10s of VMs | ~100s–1000s of containers |
| **Portability** | OS-dependent images | Run anywhere Docker runs |

### When to use which

- **VM** — running different OSes on one host, strong security boundaries (different tenants).
- **Container** — packaging apps, microservices, dev environments, CI/CD pipelines.

> Modern stacks often **combine both** — VMs as the host, containers running inside.

---

## 4. Images, Containers & Registries — The Three Big Words

You'll hear these three constantly. Get them straight once and you're set.

### Image

A **read-only template** that contains everything needed to run an app: code, runtime, libraries, config files.

**Analogy:** A **class** in programming. Or a **blueprint** for a house.

```bash
docker images          # list images on your machine
```

### Container

A **running instance** of an image. You can have many containers from the same image, just like you can have many objects from one class.

**Analogy:** An **object** instantiated from a class. Or a **house** built from a blueprint — you can build 10 identical houses from one blueprint.

```bash
docker ps              # list running containers
docker ps -a           # list all containers (including stopped)
```

### Registry

A **library** where images are stored and shared. The default is **Docker Hub** (`hub.docker.com`).

**Analogy:** **GitHub for images.** You `pull` to download, `push` to upload.

```bash
docker pull nginx                    # download image from Docker Hub
docker push myname/myapp:1.0         # upload your image
```

### The flow

```
   Registry (Docker Hub)
        │
        │  docker pull
        ▼
   Image (on your machine)
        │
        │  docker run
        ▼
   Container (running app)
```

---

## 5. Installing Docker & Your First Container

### Install

- **macOS / Windows** — [Docker Desktop](https://www.docker.com/products/docker-desktop) (free for personal use).
- **Linux** — `apt install docker.io` or follow distro instructions for **Docker Engine**.

Verify:

```bash
docker --version
docker run hello-world
```

The `hello-world` container is the "Hello, World!" of Docker — it downloads a tiny image, runs it, prints a message, and exits.

### What just happened

When you ran `docker run hello-world`:

1. Docker looked locally for the `hello-world` image — didn't find it.
2. Pulled it from Docker Hub.
3. Created a container from it.
4. Started the container.
5. The container ran its program (printed a message) and exited.

Five steps you'll repeat thousands of times.

### Your first useful container — run nginx

```bash
docker run -d -p 8080:80 --name web nginx
```

Now open `http://localhost:8080` in your browser. You're running a real web server in a container!

Flag breakdown:
- `-d` — detached mode (runs in background).
- `-p 8080:80` — map host port 8080 to container port 80.
- `--name web` — give the container a friendly name.
- `nginx` — the image to use.

Stop it:

```bash
docker stop web
docker rm web
```

---

## 6. The Docker Architecture

Docker has three main pieces working together:

```
┌────────────────────┐         ┌────────────────────────────────┐
│  Docker Client     │         │     Docker Host                 │
│  (your terminal)   │ ──API──▶│  ┌──────────────────────────┐  │
│  $ docker run ...  │         │  │   Docker Daemon (dockerd)│  │
└────────────────────┘         │  │                          │  │
                               │  │   Containers   Images    │  │
                               │  │   ┌──┐ ┌──┐   ┌──┐ ┌──┐  │  │
                               │  │   └──┘ └──┘   └──┘ └──┘  │  │
                               │  └──────────┬───────────────┘  │
                               └─────────────┼──────────────────┘
                                             │
                                             ▼
                                    ┌─────────────────┐
                                    │    Registry     │
                                    │  (Docker Hub)   │
                                    └─────────────────┘
```

### The pieces

| Piece | What it does |
|---|---|
| **Docker Client** (`docker` CLI) | Translates your commands into API calls. |
| **Docker Daemon** (`dockerd`) | The brains — manages images, containers, networks, volumes. Listens for API requests. |
| **Registry** | Where images live (Docker Hub, AWS ECR, GitHub Container Registry, private registries). |

When you type `docker run nginx`, the client sends an API call to the daemon. The daemon pulls the image (if needed), creates and starts the container.

> **Important:** On macOS and Windows, Docker actually runs a tiny Linux VM under the hood (because containers need Linux). Docker Desktop hides this complexity. On Linux, it runs natively.

---

## 7. Understanding the Dockerfile

A **Dockerfile** is a plain text file with instructions for building an image. Each line is a step.

### A simple example — Node.js app

```dockerfile
# Start from an existing image
FROM node:20-alpine

# Set the working directory inside the container
WORKDIR /app

# Copy package files first (caching trick — explained later)
COPY package*.json ./

# Install dependencies
RUN npm install

# Copy the rest of the app
COPY . .

# Tell Docker which port the app listens on
EXPOSE 3000

# The command to run when the container starts
CMD ["node", "server.js"]
```

### The main instructions

| Instruction | Purpose | Example |
|---|---|---|
| `FROM` | Base image to start from | `FROM python:3.12-slim` |
| `WORKDIR` | Set the working directory | `WORKDIR /app` |
| `COPY` | Copy files from host into image | `COPY . .` |
| `ADD` | Like COPY but supports URLs and tar extraction | `ADD https://... /tmp/` |
| `RUN` | Run a command during build | `RUN apt-get update && apt-get install -y curl` |
| `ENV` | Set an environment variable | `ENV NODE_ENV=production` |
| `EXPOSE` | Document which port the app uses | `EXPOSE 8080` |
| `CMD` | Default command when container starts | `CMD ["node", "index.js"]` |
| `ENTRYPOINT` | Fixed command; CMD becomes its args | `ENTRYPOINT ["python"]` |
| `USER` | Switch to a non-root user | `USER node` |
| `ARG` | Build-time variable | `ARG VERSION=1.0` |

### `CMD` vs `ENTRYPOINT` — the classic confusion

- **`CMD`** — the default command, but can be overridden when running the container.
- **`ENTRYPOINT`** — fixed; `CMD` becomes its arguments.

```dockerfile
ENTRYPOINT ["python"]
CMD ["app.py"]
```

- `docker run myimg` → runs `python app.py`
- `docker run myimg test.py` → runs `python test.py` (CMD overridden)

Most beginners just use `CMD` and that's fine.

---

## 8. Building Your First Image

Let's actually build something. Create a folder with these two files:

**`server.js`**
```js
const http = require('http');
const server = http.createServer((req, res) => {
  res.end('Hello from Docker!\n');
});
server.listen(3000, () => console.log('Listening on 3000'));
```

**`Dockerfile`**
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY server.js .
EXPOSE 3000
CMD ["node", "server.js"]
```

### Build the image

```bash
docker build -t my-app .
```

Breakdown:
- `build` — build an image.
- `-t my-app` — tag it with name `my-app`.
- `.` — use the current directory as the build context.

You'll see output like:
```
[+] Building 8.2s (8/8) FINISHED
 => [internal] load build definition from Dockerfile
 => [1/4] FROM docker.io/library/node:20-alpine
 => [2/4] WORKDIR /app
 => [3/4] COPY server.js .
 => [4/4] CMD ["node", "server.js"]
 => exporting to image
 => => naming to docker.io/library/my-app
```

### Run it

```bash
docker run -d -p 3000:3000 --name myapp my-app
```

Visit `http://localhost:3000` — you should see "Hello from Docker!".

You just packaged a Node.js app into an image that runs the same way on any machine in the world that has Docker. That's the magic.

---

## 9. Image Layers & Caching

Each instruction in a Dockerfile creates a **layer**. Layers are stacked and cached, which is why Docker can rebuild quickly.

### Why layers matter

```dockerfile
FROM node:20-alpine          # Layer 1
WORKDIR /app                 # Layer 2
COPY package*.json ./        # Layer 3
RUN npm install              # Layer 4 (slow!)
COPY . .                     # Layer 5
CMD ["node", "server.js"]    # Layer 6
```

When you change a file in your app and rebuild:
- Layers 1–4 are **reused from cache** (nothing changed).
- Only layer 5 onwards is rebuilt.

**This is huge.** `npm install` might take 60 seconds; you don't want to repeat it every time you change a line of code.

### The golden rule of Dockerfile ordering

**Put things that change LEAST at the top, MOST at the bottom.**

```dockerfile
# BAD — re-runs npm install on every code change!
COPY . .
RUN npm install

# GOOD — npm install only re-runs when package.json changes
COPY package*.json ./
RUN npm install
COPY . .
```

### Visualizing layers

```bash
docker history my-app
```

Shows each layer with size and the command that created it.

### Layer caching for `apt`/`apk`

```dockerfile
# BAD — these can produce different results over time
RUN apt-get update
RUN apt-get install -y curl

# GOOD — combine to ensure update + install run together
RUN apt-get update && apt-get install -y \
    curl \
    git \
    && rm -rf /var/lib/apt/lists/*
```

The `rm -rf /var/lib/apt/lists/*` cleans the apt cache to keep the image small.

---

## 10. Essential Docker Commands

Memorize this short list and you can do 90% of Docker work.

### Images

```bash
docker images                       # list images
docker pull nginx:1.25              # download an image
docker build -t myapp .             # build from Dockerfile
docker rmi myapp                    # remove image
docker tag myapp myname/myapp:1.0   # add another tag
docker push myname/myapp:1.0        # upload to registry
```

### Containers

```bash
docker run -d -p 8080:80 nginx      # run in background
docker run -it ubuntu bash          # interactive shell
docker ps                           # list running
docker ps -a                        # list all (incl stopped)
docker stop <id|name>               # graceful stop
docker start <id|name>              # start a stopped one
docker restart <id|name>            # restart
docker rm <id|name>                 # remove (must be stopped)
docker rm -f <id|name>              # force remove (kills running)
```

### Inspecting

```bash
docker logs <id|name>               # see stdout/stderr
docker logs -f <id|name>            # follow logs
docker exec -it <id|name> sh        # shell into a running container
docker inspect <id|name>            # detailed JSON info
docker stats                        # live CPU/memory of all containers
```

### Cleanup

```bash
docker system prune                 # remove unused data
docker system prune -a              # also remove unused images
docker container prune              # stopped containers
docker volume prune                 # unused volumes
docker image prune                  # dangling images
```

> **`docker system prune -a` reclaims gigabytes** and is the first thing to try when your disk fills up.

---

## 11. Container Lifecycle

A container goes through clear states:

```
       docker run
            │
            ▼
       ┌─────────┐
       │ created │
       └────┬────┘
            │ docker start
            ▼
       ┌─────────┐  docker pause   ┌─────────┐
       │ running │ ──────────────▶ │ paused  │
       │         │ ◀────────────── │         │
       └────┬────┘  docker unpause └─────────┘
            │
            │ docker stop / process exit
            ▼
       ┌─────────┐
       │ stopped │
       └────┬────┘
            │ docker rm
            ▼
        (removed)
```

### Quick mental model

- **Image** is on disk forever (until you `rmi`).
- **Container** is the running process. When it stops, it's still there (you can restart it). Only `docker rm` deletes it.
- **`docker run`** is shorthand for `create + start`.

### Detached vs attached

```bash
docker run -d nginx          # detached — runs in background, returns
docker run nginx             # attached — output streams to your terminal
docker run -it ubuntu bash   # interactive — you get a shell
```

The `-it` flag combo:
- `-i` — interactive (keep STDIN open).
- `-t` — allocate a pseudo-TTY (makes it feel like a real terminal).

---

## 12. Port Mapping — Exposing Containers to the World

Containers have their own network. By default, you can't reach a container from your host browser — you need **port mapping**.

```bash
docker run -d -p 8080:80 nginx
```

This maps **host port 8080** → **container port 80**.

```
Browser → http://localhost:8080
            │
            ▼
        [Host machine port 8080]
            │
            │  Docker forwards
            ▼
        [Container port 80] → nginx
```

### Format: `-p HOST:CONTAINER`

```bash
docker run -p 3000:3000 myapp     # same port both sides
docker run -p 8080:80 nginx       # different ports
docker run -p 80:80 -p 443:443 nginx  # multiple ports
docker run -P nginx               # auto-map all EXPOSEd ports to random host ports
```

### `EXPOSE` in Dockerfile is just documentation

```dockerfile
EXPOSE 3000
```

This doesn't actually publish the port — you still need `-p` at runtime. `EXPOSE` is metadata so others know which port the app listens on.

### Common gotcha

If you don't use `-p`, your app is running but unreachable from outside. Beginners often forget this and think Docker is broken.

---

## 13. Volumes & Persistent Data

Containers are **ephemeral** — when you remove a container, all data inside it is gone. To persist data, use **volumes** or **bind mounts**.

### Three ways to handle files

#### 1. Container filesystem (default)

Data lives in the container. **Lost when container is removed.** Fine for stateless apps.

#### 2. Bind mount — sync a host folder

Mount a folder from your host into the container.

```bash
docker run -v /Users/abhi/myapp:/app node:20 node /app/server.js
```

The container sees `/app` as the host's `/Users/abhi/myapp`. Changes on either side are visible to both. **Great for development** — edit on host, see changes inside container.

#### 3. Named volume — managed by Docker

```bash
docker volume create mydata
docker run -v mydata:/var/lib/postgresql/data postgres
```

Docker manages the storage location. Data survives container removal.

```bash
docker volume ls          # list volumes
docker volume inspect mydata
docker volume rm mydata
```

### When to use which

| Scenario | Use |
|---|---|
| Development — edit code on host, run in container | **Bind mount** |
| Database data (Postgres, MySQL, Mongo) | **Named volume** |
| Stateless app, no persistence needed | **Nothing** (container FS) |
| Sharing data between containers | **Named volume** |

### Practical example — Postgres with persistent storage

```bash
docker run -d \
  --name pg \
  -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:16
```

Stop and remove the container — your data is safe in the `pgdata` volume. Start a new container with the same `-v pgdata:...` and your data is back.

---

## 14. Environment Variables & Configuration

Never hardcode config (DB URLs, API keys, ports) into your image. Pass them in as env vars.

### Set env vars at runtime

```bash
docker run -e NODE_ENV=production -e DB_URL=postgres://... myapp
```

### Use a file for many vars

Create a `.env` file:
```
NODE_ENV=production
DB_URL=postgres://user:pass@db:5432/mydb
JWT_SECRET=supersecret
```

```bash
docker run --env-file .env myapp
```

### Set defaults in the Dockerfile

```dockerfile
ENV NODE_ENV=production
ENV PORT=3000
```

These act as defaults — `-e` at runtime overrides them.

### Reading env vars in your app

```js
// Node
const port = process.env.PORT || 3000;
const dbUrl = process.env.DB_URL;
```

```python
# Python
import os
port = int(os.getenv('PORT', '3000'))
```

### Security note

`-e` is fine for non-secret config. For real secrets in production, use:
- **Docker Secrets** (with Swarm).
- **Kubernetes Secrets** + external secret stores.
- **AWS Secrets Manager / HashiCorp Vault** + read at startup.

Never bake secrets into images — they end up in registries and Git history.

---

## 15. Docker Networking

Containers can talk to each other through Docker networks.

### Three default network types

| Network | What it does |
|---|---|
| **bridge** (default) | Containers on the same bridge can talk to each other by IP |
| **host** | Container uses the host's network directly (no isolation) |
| **none** | Container has no network |

### The big upgrade — custom bridge networks

By default, containers on the default `bridge` can talk by IP but **not by name**. Create your own network and they can resolve each other by container name:

```bash
docker network create myapp-net

docker run -d --name db --network myapp-net postgres
docker run -d --name api --network myapp-net myapp

# Inside `api`, you can connect to `postgres://db:5432/...`
# The hostname `db` resolves to the container automatically!
```

This is **the** way to wire up multi-container apps.

### Common commands

```bash
docker network ls
docker network create mynet
docker network inspect mynet
docker network rm mynet
docker network connect mynet mycontainer
```

### Practical example — Node app + Postgres

```bash
docker network create app-net

# Database
docker run -d \
  --name db \
  --network app-net \
  -e POSTGRES_PASSWORD=secret \
  postgres:16

# App — connects to DB using hostname "db"
docker run -d \
  --name api \
  --network app-net \
  -e DB_URL=postgres://postgres:secret@db:5432/postgres \
  -p 3000:3000 \
  myapp
```

The API can reach the DB as `db:5432` — no IPs hardcoded.

---

## 16. Docker Compose — Multi-Container Apps Made Easy

Running `docker run` for every container with all its flags gets old fast. **Docker Compose** lets you declare your whole stack in one YAML file and run it with one command.

### Example — Node app + Postgres + Redis

Create `docker-compose.yml`:

```yaml
services:
  api:
    build: .
    ports:
      - "3000:3000"
    environment:
      DB_URL: postgres://postgres:secret@db:5432/postgres
      REDIS_URL: redis://cache:6379
    depends_on:
      - db
      - cache

  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - pgdata:/var/lib/postgresql/data

  cache:
    image: redis:7-alpine

volumes:
  pgdata:
```

Start everything:

```bash
docker compose up -d
```

Stop everything:

```bash
docker compose down
```

That single file replaces:
- 3 networks/configs.
- 3 `docker run` commands with many flags.
- Manual ordering.

### Useful Compose commands

```bash
docker compose up                   # start (foreground)
docker compose up -d                # start (background)
docker compose down                 # stop and remove containers
docker compose down -v              # also remove volumes
docker compose ps                   # list services
docker compose logs -f api          # tail logs of one service
docker compose exec api sh          # shell into a service
docker compose build                # rebuild images
docker compose restart api          # restart one service
```

### Why Compose is great for development

- **One file** describes your whole local environment.
- **Networks** are auto-created — services can reach each other by name.
- **Volumes** are managed.
- **One command** starts everything.

> For production, Compose is fine for small setups. For real scale, you'd use Kubernetes — but Compose remains the gold standard for local development.

---

## 17. Multi-Stage Builds

A common problem: your build needs heavy tools (compilers, dev dependencies), but the final image should be small. **Multi-stage builds** fix this — use one image to build, then copy artifacts to a smaller final image.

### Single-stage (bad — bloated)

```dockerfile
FROM node:20
WORKDIR /app
COPY . .
RUN npm install
RUN npm run build
CMD ["node", "dist/server.js"]
```

Final image: ~1.2 GB (full Node + all dev deps + source code).

### Multi-stage (good — lean)

```dockerfile
# Stage 1: build
FROM node:20 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Stage 2: runtime (small!)
FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY package*.json ./
CMD ["node", "dist/server.js"]
```

Final image: ~150 MB. Only the built output + runtime deps.

### Example — Go (even better)

```dockerfile
# Stage 1: compile
FROM golang:1.22 AS builder
WORKDIR /src
COPY . .
RUN go build -o /app/server

# Stage 2: minimal runtime
FROM alpine:3.19
COPY --from=builder /app/server /app/server
CMD ["/app/server"]
```

Final image: ~15 MB. Just Alpine + a single Go binary.

### Why this matters

Smaller images mean:
- **Faster pulls** in CI/CD.
- **Faster deploys** to production.
- **Less attack surface** (fewer installed packages = fewer CVEs).
- **Lower storage costs** in registries.

---

## 18. Image Tags & Versioning

Every image has a **name** and a **tag**:

```
nginx:1.25-alpine
│      │
name   tag
```

- No tag → defaults to `latest`.
- Tags are just labels — they can point to different image versions over time.

### Tagging your own images

```bash
docker build -t myapp:1.0 .
docker build -t myapp:latest -t myapp:1.0 -t myapp:1.0.3 .
```

### `latest` is a trap

`latest` is just a default name — it does NOT automatically point to the newest version. If you publish `myapp:2.0` but don't also tag it `latest`, then `myapp:latest` keeps pointing at the old version.

**In production, never use `latest`.** Pin to a specific version:

```bash
docker pull node:20.11.1-alpine    # ✅ reproducible
docker pull node:latest            # ❌ different result every week
```

Better: pin to a digest for true immutability:

```bash
docker pull nginx@sha256:abc123def456...
```

### Semantic versioning convention

```
myapp:1
myapp:1.2
myapp:1.2.3
myapp:1.2.3-alpine
myapp:1.2.3-debian
```

Users can pin to whatever specificity they want.

---

## 19. Docker Hub & Other Registries

**Docker Hub** (`hub.docker.com`) is the default public registry. Anyone can pull public images; you need an account to push.

### Login

```bash
docker login                        # logs into Docker Hub
docker login ghcr.io                # GitHub Container Registry
docker login <accountid>.dkr.ecr.us-east-1.amazonaws.com  # AWS ECR
```

### Pushing an image

```bash
docker build -t myname/myapp:1.0 .
docker push myname/myapp:1.0
```

The name format `<username>/<image>:<tag>` is what tells Docker Hub where it belongs.

### Common registries

| Registry | URL | Use case |
|---|---|---|
| **Docker Hub** | `docker.io` | Public sharing, the default |
| **GitHub Container Registry** | `ghcr.io` | If your code is on GitHub |
| **AWS ECR** | `*.dkr.ecr.<region>.amazonaws.com` | AWS deployments |
| **Google Artifact Registry** | `*.pkg.dev` | GCP deployments |
| **Azure Container Registry** | `*.azurecr.io` | Azure deployments |
| **Self-hosted (Harbor, Nexus)** | Your own | Enterprise, air-gapped |

### Public vs private

- **Public** — anyone can pull. Fine for open source. Free.
- **Private** — requires authentication. Costs a small fee on Docker Hub; usually free with cloud providers up to some quota.

---

## 20. `.dockerignore` & Build Context

When you run `docker build .`, Docker sends the **entire current directory** to the daemon as the "build context". Big folders = slow builds and potential leakage of secrets into the image.

### `.dockerignore` — like `.gitignore` for Docker

Create a `.dockerignore` file:

```
node_modules
.git
.env
.env.*
*.log
dist
coverage
.DS_Store
README.md
.vscode
.idea
```

This excludes those files/folders from the build context — faster builds and you won't accidentally ship secrets.

### Why this matters

- **`node_modules`** — often hundreds of MB; you'll reinstall inside the container anyway.
- **`.git`** — sensitive history, not needed at runtime.
- **`.env`** — secrets! Never put these in your image.

> A missing `.dockerignore` is one of the most common Docker mistakes. Always add one.

---

## 21. Logs, Exec, Inspect — Debugging Containers

When a container misbehaves, three tools solve 90% of issues.

### `docker logs` — see stdout/stderr

```bash
docker logs myapp                # all logs
docker logs -f myapp             # follow (live)
docker logs --tail 100 myapp     # last 100 lines
docker logs --since 10m myapp    # last 10 minutes
```

Apps in containers should log to **stdout/stderr** (not files). Docker captures these automatically.

### `docker exec` — run commands inside a container

```bash
docker exec -it myapp sh         # shell into running container
docker exec -it myapp bash       # bash if available
docker exec myapp ls /app        # run a single command
docker exec myapp env            # see env vars
```

This is your debugging swiss army knife — get inside and look around.

### `docker inspect` — detailed info

```bash
docker inspect myapp
```

Outputs JSON with everything: networks, mounts, env vars, IPs, command, restart policy, etc.

Useful filters:
```bash
docker inspect -f '{{.NetworkSettings.IPAddress}}' myapp
docker inspect -f '{{.State.Status}}' myapp
```

### `docker stats` — live resource usage

```bash
docker stats                     # CPU, memory, network for all running
docker stats myapp               # one container
```

### `docker events` — stream Docker daemon events

```bash
docker events                    # see container start/stop/die in real time
```

---

## 22. Security Basics

Containers aren't magically secure. The basics:

### 1. Don't run as root

By default, containers run as root. If an attacker escapes, they're root on your host (in extreme cases).

```dockerfile
# Create a non-root user
RUN addgroup -S app && adduser -S app -G app
USER app
```

### 2. Use minimal base images

Smaller image = fewer packages = fewer vulnerabilities.

- `alpine` (~5 MB) — minimal Linux.
- `distroless` (Google) — just your app and runtime; no shell, no package manager.
- `scratch` — empty; for static binaries (Go, Rust).

### 3. Pin versions and digests

```dockerfile
# ❌ Could change tomorrow
FROM node

# ✅ Specific version
FROM node:20.11.1-alpine

# ✅ Truly immutable
FROM node:20.11.1-alpine@sha256:abc123...
```

### 4. Scan for vulnerabilities

```bash
docker scout cves myapp:1.0      # built-in scanner
trivy image myapp:1.0            # popular alternative
```

Most CI/CD systems support this — fail the build on high-severity CVEs.

### 5. Don't bake secrets into images

Even `RUN echo $SECRET > /file` ends up in a layer. Use:
- Build secrets: `--secret` flag (BuildKit).
- Runtime: env vars from secret stores.

### 6. Read-only root filesystem

```bash
docker run --read-only myapp
```

App can't write to its own filesystem — limits damage from many attacks.

### 7. Drop unnecessary capabilities

```bash
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myapp
```

By default, containers have many Linux capabilities. Drop what you don't need.

---

## 23. Production Best Practices

A checklist for production-ready images:

- [ ] **Use a specific base image tag** — never `:latest`.
- [ ] **Multi-stage build** to minimize final image size.
- [ ] **Non-root user** (`USER` directive).
- [ ] **`.dockerignore`** present and comprehensive.
- [ ] **`HEALTHCHECK`** for liveness:
   ```dockerfile
   HEALTHCHECK --interval=30s --timeout=3s \
     CMD curl -f http://localhost:3000/health || exit 1
   ```
- [ ] **Logs go to stdout/stderr** — not log files.
- [ ] **Single process per container** — easier to monitor and restart.
- [ ] **Use ENV for config**, not baked-in.
- [ ] **Image scanned** for CVEs in CI.
- [ ] **Resource limits** set when running:
  ```bash
  docker run --memory=512m --cpus=1.0 myapp
  ```
- [ ] **Restart policy** set:
  ```bash
  docker run --restart=unless-stopped myapp
  ```
- [ ] **Layers ordered for cache efficiency** (changing files at the bottom).

### Build with BuildKit

BuildKit is the modern, faster build engine. Enabled by default in newer Docker. Lets you do parallel builds, secret mounts, and cache mounts:

```dockerfile
# syntax=docker/dockerfile:1.6
FROM node:20-alpine
RUN --mount=type=cache,target=/root/.npm \
    npm install   # cache persists across builds
```

---

## 24. Common Pitfalls & Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| "Cannot connect to Docker daemon" | Daemon not running | Start Docker Desktop / `sudo systemctl start docker` |
| App not reachable from browser | No `-p` flag | Add `-p HOST:CONTAINER` |
| Changes to host files don't show in container | No volume mount | Use `-v $(pwd):/app` |
| Image rebuild is slow every time | Bad layer order | Copy `package.json` first, then install, then code |
| Image is huge | Single-stage with dev tools | Use multi-stage build |
| `latest` tag changed unexpectedly | Mutable tag | Pin to specific version or digest |
| Container exits immediately | App finished or errored | `docker logs <name>` to see why |
| `port is already allocated` | Another container/process using the port | Stop it or use a different host port |
| `no space left on device` | Old images/containers/volumes | `docker system prune -a` |
| Code change doesn't reflect | Image cached | Rebuild with `--no-cache` or use bind mounts in dev |
| Env vars not visible | Used `ENV` after copying app? | `ENV` after `COPY` is fine; check spelling |
| `permission denied` writing to mounted volume | UID mismatch | Set matching UID in Dockerfile or use named volume |

### Debugging workflow

```bash
# 1. Did the container start?
docker ps -a

# 2. If it crashed, what did it say?
docker logs <name>

# 3. If it's running but broken, get inside
docker exec -it <name> sh

# 4. Check env vars, files, network
env
ls /app
ping db

# 5. Check resource usage
docker stats
```

---

## 25. Practical Use Cases

### Use case 1 — Consistent dev environment

A new team member joins. Instead of "install Node 20.11, install Postgres 16, set up Redis, configure...", they run:

```bash
git clone repo
docker compose up
```

Done. Their environment matches everyone else's exactly.

### Use case 2 — Running databases locally

No need to install Postgres, MySQL, Mongo on your machine:

```bash
docker run -d --name pg -e POSTGRES_PASSWORD=secret -p 5432:5432 postgres:16
docker run -d --name mysql -e MYSQL_ROOT_PASSWORD=secret -p 3306:3306 mysql:8
docker run -d --name redis -p 6379:6379 redis:7
```

Each project can use a different version without conflict. `docker rm -f` and they're gone.

### Use case 3 — Trying new tools risk-free

Want to try a database, a programming language, a tool?

```bash
docker run -it --rm python:3.12 python
docker run -it --rm ubuntu bash
docker run -it --rm node:20 node
```

`--rm` auto-deletes the container when it exits. Try, learn, throw away.

### Use case 4 — CI/CD pipelines

Build, test, and deploy in identical environments. GitHub Actions, GitLab CI, CircleCI all use Docker internally:

```yaml
# .github/workflows/test.yml
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t myapp .
      - run: docker run myapp npm test
```

### Use case 5 — Microservices

Each service in its own container, talking over a Docker network:

```yaml
services:
  auth:    { build: ./auth,    ports: ["3001:3000"] }
  users:   { build: ./users,   ports: ["3002:3000"] }
  orders:  { build: ./orders,  ports: ["3003:3000"] }
  gateway: { build: ./gateway, ports: ["80:80"] }
  db:      { image: postgres:16 }
  cache:   { image: redis:7 }
```

`docker compose up` and you have a microservice setup running locally.

### Use case 6 — Frontend build artifacts

Build a React app in one stage, serve with nginx in another:

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

Final image: ~30 MB. Static files served by tiny nginx. Perfect for production.

---

## 26. Docker vs Kubernetes — Where Each Fits

A common confusion: aren't these competitors? No — they solve different problems.

| | Docker | Kubernetes |
|---|---|---|
| **What it is** | Tool for building and running containers | Orchestrator for running containers at scale |
| **Scope** | Single host (your laptop, one server) | Cluster of many machines |
| **Solves** | "How do I package this app?" | "How do I run 1000 containers across 50 servers with healing, scaling, networking?" |
| **You use it for** | Local dev, CI/CD, building images | Production deployment of microservices |

**Analogy:** Docker is like making one really good lunchbox. Kubernetes is the catering company that distributes thousands of lunchboxes across a stadium, handles complaints, refills empty ones, and schedules deliveries.

### How they fit together

```
   Dockerfile → docker build → Image
                                 │
                                 ▼
                        Push to registry
                                 │
                                 ▼
                    Kubernetes pulls image
                                 │
                                 ▼
              Runs containers across cluster
```

You use Docker to build images. Kubernetes uses those images to run them at scale.

> If you've understood this whole guide, you have a great foundation for learning Kubernetes — see [kubernetes-concepts.md](./kubernetes-concepts.md).

---

## 27. Interview Q&A

**Q: What is Docker in one sentence?**
A tool to package applications with all their dependencies into portable containers that run the same way on any machine.

**Q: Difference between image and container?**
Image is the blueprint (read-only template). Container is a running instance of an image. One image, many containers.

**Q: Difference between a container and a VM?**
VMs virtualize the whole OS (heavy, GBs, slow boot). Containers share the host kernel and only package the app + libraries (lightweight, MBs, sub-second boot).

**Q: What is a Dockerfile?**
A text file with instructions for building an image — base image, files to copy, commands to run, default startup command.

**Q: What's the difference between CMD and ENTRYPOINT?**
ENTRYPOINT is the fixed command. CMD provides default arguments. CMD alone can be fully overridden at runtime; ENTRYPOINT can't (without `--entrypoint`).

**Q: What is image layering?**
Each Dockerfile instruction creates a cached layer. Unchanged layers are reused, making rebuilds fast. Put rarely-changing things at the top.

**Q: How do you persist data across container restarts?**
Use a volume (`-v name:/path` or `-v /host/path:/container/path`). Without a volume, data is lost when the container is removed.

**Q: What's a bind mount vs a named volume?**
Bind mount syncs a host folder into the container — great for dev. Named volume is managed by Docker, ideal for production data (databases).

**Q: How do containers talk to each other?**
Put them on the same Docker network. They can resolve each other by container name.

**Q: What does `-p 8080:80` mean?**
Map host port 8080 to container port 80. External traffic to `localhost:8080` hits port 80 inside the container.

**Q: What is Docker Compose?**
A tool to define multi-container apps in a single YAML file (`docker-compose.yml`) and start them all with `docker compose up`.

**Q: What is a multi-stage build?**
A Dockerfile with multiple `FROM` instructions. Use one stage to build (heavy tools) and copy artifacts to a small final stage. Massively reduces image size.

**Q: Why shouldn't you use `:latest` in production?**
`latest` is just a label, not a version. It can point to different images over time, breaking reproducibility. Pin to a specific version or digest.

**Q: How do you reduce Docker image size?**
Use small base images (alpine, distroless), multi-stage builds, combine RUN commands, clean apt/npm caches, use `.dockerignore`.

**Q: What is `.dockerignore`?**
Like `.gitignore` — excludes files from the build context. Speeds builds and prevents accidentally shipping secrets or junk.

**Q: How do you pass secrets to containers?**
Env vars for non-secret config. For real secrets, use Docker Secrets (Swarm), Kubernetes Secrets + external secret store, or read from a vault at startup. Never bake into images.

**Q: How do you debug a container that won't start?**
`docker logs <name>` to see error output. `docker inspect` for full config. `docker run -it <image> sh` to get a shell on a fresh container and try things manually.

**Q: What happens when you run `docker run nginx`?**
Docker pulls the image if not present, creates a container from it, starts the container, and runs the default command (nginx). The container stays running as long as nginx runs.

**Q: How does Docker work on macOS / Windows since containers need Linux?**
Docker Desktop runs a tiny Linux VM under the hood. The CLI on your Mac talks to the daemon inside that VM. You don't see it because Docker Desktop manages it.

**Q: What is the Docker daemon?**
The background process (`dockerd`) that manages images, containers, volumes, and networks. The CLI sends API requests to the daemon.

**Q: When would you NOT use Docker?**
Single static binary that doesn't need dependencies. Apps requiring kernel-level access or specific hardware. Performance-critical workloads where shared-kernel isolation matters. Tiny scripts where Docker overhead exceeds the app.

---

## 28. Quick Command Cheat Sheet

```bash
# IMAGES
docker images                              # list
docker pull <image>                        # download
docker rmi <image>                         # remove
docker build -t name:tag .                 # build from Dockerfile
docker build --no-cache -t name .          # fresh build
docker tag old new                         # add another tag
docker push name:tag                       # upload to registry
docker history <image>                     # show layers
docker save -o file.tar <image>            # export to tar
docker load -i file.tar                    # import from tar

# CONTAINERS
docker run <image>                         # run
docker run -d <image>                      # detached
docker run -it <image> sh                  # interactive shell
docker run --rm <image>                    # auto-remove on exit
docker run -p 8080:80 <image>              # port mapping
docker run -v vol:/path <image>            # volume
docker run -e KEY=value <image>            # env var
docker run --name myapp <image>            # named
docker run --restart=unless-stopped <image>  # auto-restart
docker run --network mynet <image>         # custom network
docker ps                                  # list running
docker ps -a                               # list all
docker start/stop/restart <name>
docker rm <name>                           # remove (stopped)
docker rm -f <name>                        # force remove

# INSPECTION
docker logs <name>                         # logs
docker logs -f <name>                      # follow
docker exec -it <name> sh                  # shell in
docker inspect <name>                      # full JSON info
docker stats                               # live resource usage
docker top <name>                          # processes in container

# VOLUMES
docker volume create <name>
docker volume ls
docker volume inspect <name>
docker volume rm <name>

# NETWORKS
docker network create <name>
docker network ls
docker network connect <net> <container>
docker network rm <name>

# CLEANUP
docker system prune                        # unused data
docker system prune -a                     # also unused images
docker container prune                     # stopped containers
docker image prune                         # dangling images
docker volume prune                        # unused volumes
docker system df                           # show disk usage

# COMPOSE
docker compose up                          # start
docker compose up -d                       # detached
docker compose down                        # stop & remove
docker compose down -v                     # also remove volumes
docker compose ps                          # list services
docker compose logs -f <service>           # follow logs
docker compose exec <service> sh           # shell into service
docker compose build                       # rebuild images
docker compose restart <service>           # restart one
docker compose pull                        # pull latest images

# REGISTRY
docker login                               # docker hub
docker login ghcr.io                       # github
docker push username/image:tag
docker pull username/image:tag
```

---

> **Final advice for beginners:** Don't try to learn Docker by reading. Pick a small app you've built (or any tutorial app) and *Dockerize it*. Write a Dockerfile, build it, run it, expose a port, add a volume, then add Postgres next to it with Compose. Five hours of hands-on will teach you more than five days of reading. Then keep going — most production engineers use containers daily, and the fluency compounds.
