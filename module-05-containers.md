# Module 05: Containers — Docker & Podman

> Part of the [DevOps Career Course](./README.md) by UncleJS

---

## Table of Contents

- [Overview](#overview)
- [Learning Objectives](#learning-objectives)
- [Beginner: What Are Containers?](#beginner-what-are-containers)
- [Beginner: Docker vs Podman — Side-by-Side](#beginner-docker-vs-podman--side-by-side)
- [Beginner: Installing Docker & Podman](#beginner-installing-docker--podman)
- [Beginner: Images & Containers](#beginner-images--containers)
- [Beginner: Running Containers](#beginner-running-containers)
- [Beginner: Writing Dockerfiles & Containerfiles](#beginner-writing-dockerfiles--containerfiles)
- [Beginner: Container Networking](#beginner-container-networking)
- [Beginner: Volumes & Persistent Storage](#beginner-volumes--persistent-storage)
- [Intermediate: Docker Compose & Podman Compose](#intermediate-docker-compose--podman-compose)
- [Intermediate: Container Registries](#intermediate-container-registries)
- [Intermediate: Podman Rootless & systemd Integration](#intermediate-podman-rootless--systemd-integration)
- [Intermediate: Container Security](#intermediate-container-security)
- [Intermediate: Image Optimization Best Practices](#intermediate-image-optimization-best-practices)
- [Tools & Commands Reference](#tools--commands-reference)
- [Hands-On Labs](#hands-on-labs)
- [Further Reading](#further-reading)

---

## Overview

Containers have transformed how software is built and deployed. They solve the "works on my machine" problem by packaging an application and all its dependencies into a single portable unit that runs identically everywhere — from a developer's laptop to a production cloud server.

This module covers both **Docker** and **Podman** with equal depth. Docker is the industry standard and most widely known. Podman is a daemonless, rootless alternative that integrates natively with Linux systemd and is increasingly preferred in security-conscious and enterprise environments.

[↑ Back to TOC](#table-of-contents)

---

## Learning Objectives

By the end of this module you will be able to:

- Explain what containers are and how they differ from VMs
- Pull, run, stop, and remove containers with both Docker and Podman
- Write Dockerfiles and Containerfiles to build custom images
- Configure container networking and persistent storage
- Use Docker Compose and Podman Compose for multi-container applications
- Push and pull images to/from container registries
- Run Podman in rootless mode and integrate containers with systemd
- Apply container security best practices
- Optimize images for size and build speed

[↑ Back to TOC](#table-of-contents)

---

## Beginner: What Are Containers?

### Containers vs Virtual Machines

| Feature | Virtual Machine | Container |
|---|---|---|
| Isolation | Full OS kernel | Linux namespaces + cgroups |
| Size | GBs | MBs |
| Startup time | Minutes | Seconds (or milliseconds) |
| Resource overhead | High (full OS) | Low (shares host kernel) |
| Portability | OS-dependent | Runs anywhere with a container runtime |
| Use case | Full OS isolation | Application packaging and delivery |

### How Containers Work

Containers use two core Linux kernel features:

- **Namespaces** — isolate: PID, network, filesystem, users, hostname
- **cgroups (Control Groups)** — limit: CPU, memory, disk I/O

```
Host OS Kernel
├── Container A (its own PID namespace, network namespace, /proc)
├── Container B (isolated from A)
└── Container C (isolated from A and B)
```

All containers share the **same host kernel** — this is why they're lightweight.

[↑ Back to TOC](#table-of-contents)

---

## Beginner: Docker vs Podman — Side-by-Side

| Feature | Docker | Podman |
|---|---|---|
| Architecture | Client + daemon (`dockerd`) | Daemonless (fork/exec model) |
| Root required | Yes (daemon runs as root) | No (rootless by default) |
| systemd integration | Limited | Native (`podman generate systemd`) |
| CLI compatibility | `docker` | `podman` (drop-in replacement) |
| Compose support | `docker compose` | `podman-compose` |
| Image format | OCI-compatible | OCI-compatible |
| Pod support | No | Yes (like Kubernetes pods) |
| Security posture | Root daemon = attack surface | Rootless = smaller attack surface |
| Prevalence | Industry standard | Growing, default in RHEL/Fedora |

> **Key Insight**: Podman's CLI is intentionally compatible with Docker. In most cases, you can replace `docker` with `podman` in a command and it works identically.

```bash
# These are equivalent:
docker run -it ubuntu:24.04 bash
podman run -it ubuntu:24.04 bash
```

[↑ Back to TOC](#table-of-contents)

---

## Beginner: Installing Docker & Podman

### Docker (Ubuntu)

```bash
# Remove old versions
sudo apt remove docker docker-engine docker.io containerd runc

# Install via official script (quickest)
curl -fsSL https://get.docker.com | sh

# Add your user to the docker group (avoid using sudo)
sudo usermod -aG docker $USER
newgrp docker

# Verify
docker version
docker run hello-world
```

### Podman (Ubuntu)

```bash
sudo apt update
sudo apt install -y podman

# Verify
podman version
podman run hello-world

# Rootless setup (usually automatic on modern systems)
podman info | grep -A2 "rootless"
```

### Podman (RHEL/Rocky/Fedora)

```bash
sudo dnf install -y podman podman-compose
```

[↑ Back to TOC](#table-of-contents)

---

## Beginner: Images & Containers

### The Container Lifecycle

```
Image (blueprint) → Container (running instance)
       │                    │
  docker build          docker run
  podman build          podman run
```

- An **image** is a read-only layered filesystem (built from a Dockerfile)
- A **container** is a running instance of an image with a writable layer on top
- Multiple containers can run from the same image simultaneously

### Working with Images

```bash
# Docker
docker pull nginx:1.25              # Pull image from Docker Hub
docker images                       # List local images
docker image ls                     # Same as above
docker image rm nginx:1.25          # Remove an image
docker image prune                  # Remove dangerously unused images
docker image inspect nginx:1.25     # Full image metadata

# Podman (identical commands)
podman pull nginx:1.25
podman images
podman image ls
podman image rm nginx:1.25
podman image prune
podman image inspect nginx:1.25
```

[↑ Back to TOC](#table-of-contents)

---

## Beginner: Running Containers

### Core Run Flags

```bash
# Run a container — Docker and Podman share the same flags
docker run nginx                     # Run in foreground (blocking)
docker run -d nginx                  # Run detached (background)
docker run -it ubuntu:24.04 bash     # Interactive with terminal
docker run --name webserver nginx    # Give it a name
docker run -p 8080:80 nginx          # Map host port 8080 → container port 80
docker run -p 127.0.0.1:8080:80 nginx  # Bind to localhost only
docker run -e ENV_VAR=value nginx    # Set environment variable
docker run -v /host/path:/container/path nginx  # Mount a volume
docker run --rm nginx                # Auto-remove container when stopped
docker run -d --restart always nginx # Always restart if it crashes

# Same with Podman:
podman run -d -p 8080:80 --name webserver nginx
```

### Managing Running Containers

```bash
# Docker
docker ps                           # List running containers
docker ps -a                        # List all containers (including stopped)
docker stop webserver               # Gracefully stop
docker start webserver              # Start a stopped container
docker restart webserver            # Stop + start
docker rm webserver                 # Remove a stopped container
docker rm -f webserver              # Force remove (even if running)
docker logs webserver               # View container logs
docker logs -f webserver            # Follow logs in real time
docker exec -it webserver bash      # Open shell in running container
docker inspect webserver            # Full container metadata

# Podman (identical):
podman ps
podman ps -a
podman stop webserver
podman logs -f webserver
podman exec -it webserver bash
podman inspect webserver
```

[↑ Back to TOC](#table-of-contents)

---

## Beginner: Writing Dockerfiles & Containerfiles

Podman uses the identical `Containerfile` format (can also use `Dockerfile` — same thing).

### Anatomy of a Dockerfile

```dockerfile
# Base image — always start with an official image
FROM ubuntu:24.04

# Set the maintainer label
LABEL maintainer="your@email.com"

# Set environment variables
ENV APP_PORT=8080
ENV APP_ENV=production

# Run commands to install dependencies
RUN apt-get update && apt-get install -y \
    python3 \
    python3-pip \
    && rm -rf /var/lib/apt/lists/*

# Set the working directory
WORKDIR /app

# Copy dependency files first (layer caching optimization)
COPY requirements.txt .
RUN pip3 install -r requirements.txt

# Copy application code
COPY . .

# Create a non-root user for security
RUN useradd -r -s /bin/false appuser
USER appuser

# Document which port the app listens on
EXPOSE 8080

# Default command to run
CMD ["python3", "app.py"]
```

### Building Images

```bash
# Docker
docker build -t myapp:1.0 .                   # Build from Dockerfile in current dir
docker build -t myapp:1.0 -f Dockerfile.prod .  # Specify a Dockerfile
docker build --no-cache -t myapp:1.0 .         # Build without cache
docker build --build-arg VERSION=1.5 -t myapp . # Pass build arguments

# Podman
podman build -t myapp:1.0 .
podman build -t myapp:1.0 -f Containerfile .
```

### Multi-Stage Builds (Advanced)

Multi-stage builds produce tiny final images by building in one stage and copying only artifacts to the final image.

```dockerfile
# Stage 1: Build
FROM golang:1.22 AS builder
WORKDIR /build
COPY . .
RUN go build -o server .

# Stage 2: Final image (much smaller)
FROM alpine:3.19
WORKDIR /app
COPY --from=builder /build/server .
EXPOSE 8080
CMD ["./server"]
```

[↑ Back to TOC](#table-of-contents)

---

## Beginner: Container Networking

### Docker Network Types

| Network | Description | Use Case |
|---|---|---|
| **bridge** | Default; containers can communicate via container name | Single-host development |
| **host** | Container shares host network stack | Performance-critical apps |
| **none** | No networking | Fully isolated workloads |
| **overlay** | Multi-host networking (Docker Swarm) | Distributed production |

```bash
# Docker
docker network ls                               # List networks
docker network create mynetwork                 # Create a custom network
docker network inspect mynetwork               # Inspect network
docker run -d --network mynetwork --name app nginx   # Connect to network

# Containers on the same custom network can reach each other by name
docker exec app curl http://database:5432      # "database" = container name

# Podman
podman network ls
podman network create mynetwork
podman run -d --network mynetwork --name app nginx
```

### Exposing Ports

```bash
# Map container port 80 to host port 8080
docker run -d -p 8080:80 nginx
podman run -d -p 8080:80 nginx

# Access: http://localhost:8080 → container port 80
```

[↑ Back to TOC](#table-of-contents)

---

## Beginner: Volumes & Persistent Storage

Containers are ephemeral — data written inside a container is lost when the container is removed. Volumes provide persistent storage.

### Volume Types

| Type | Syntax | Managed By |
|---|---|---|
| **Named volume** | `-v mydata:/app/data` | Docker/Podman |
| **Bind mount** | `-v /host/path:/container/path` | You (host filesystem) |
| **tmpfs** | `--tmpfs /tmp` | RAM (temporary, in-memory) |

```bash
# Docker — named volumes
docker volume create mydata                         # Create a volume
docker volume ls                                    # List volumes
docker volume inspect mydata                        # Inspect volume
docker run -d -v mydata:/var/lib/mysql mysql:8.0   # Use volume
docker volume rm mydata                             # Remove volume

# Bind mount — map a host directory
docker run -d -v /home/user/html:/usr/share/nginx/html nginx
podman run -d -v /home/user/html:/usr/share/nginx/html:Z nginx
# Note: Podman uses :Z or :z suffix on SELinux systems to relabel

# Podman volumes
podman volume create mydata
podman volume ls
podman run -d -v mydata:/var/lib/mysql docker.io/library/mysql:8.0
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Docker Compose & Podman Compose

Compose files define and run multi-container applications with a single command.

### docker-compose.yml / compose.yaml

```yaml
version: "3.9"

services:
  # Web application
  web:
    build: .
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgresql://postgres:password@db:5432/appdb
      - REDIS_URL=redis://cache:6379
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_started
    networks:
      - appnet
    restart: unless-stopped

  # PostgreSQL database
  db:
    image: postgres:16
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - appnet
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Redis cache
  cache:
    image: redis:7-alpine
    networks:
      - appnet

volumes:
  pgdata:

networks:
  appnet:
    driver: bridge
```

### Compose Commands

```bash
# Docker Compose (v2 — built into docker CLI)
docker compose up -d                # Start all services in background
docker compose down                 # Stop and remove containers
docker compose down -v              # Also remove volumes
docker compose ps                   # Show status of services
docker compose logs -f              # Follow all service logs
docker compose logs -f web          # Follow logs of one service
docker compose exec web bash        # Open shell in running service
docker compose build                # Rebuild images
docker compose pull                 # Pull latest images
docker compose restart web          # Restart one service

# Podman Compose
podman-compose up -d
podman-compose down
podman-compose ps
podman-compose logs -f
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Container Registries

A registry stores and distributes container images.

| Registry | URL | Notes |
|---|---|---|
| Docker Hub | `docker.io` | Default public registry |
| GitHub Container Registry | `ghcr.io` | Integrated with GitHub |
| AWS ECR | `<account>.dkr.ecr.<region>.amazonaws.com` | AWS-managed |
| Azure ACR | `<name>.azurecr.io` | Azure-managed |
| Google Artifact Registry | `<region>-docker.pkg.dev` | GCP-managed |
| Quay.io | `quay.io` | Red Hat managed |

```bash
# Login to registries
docker login                                    # Docker Hub
docker login ghcr.io -u USERNAME               # GitHub Container Registry
docker login 123456789.dkr.ecr.us-east-1.amazonaws.com  # AWS ECR

# Podman login
podman login docker.io
podman login ghcr.io -u USERNAME

# Tag and push an image
docker tag myapp:1.0 ghcr.io/uncleJS/myapp:1.0
docker push ghcr.io/uncleJS/myapp:1.0

podman tag myapp:1.0 ghcr.io/uncleJS/myapp:1.0
podman push ghcr.io/uncleJS/myapp:1.0

# Pull from a specific registry
docker pull ghcr.io/uncleJS/myapp:1.0
podman pull ghcr.io/uncleJS/myapp:1.0
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Podman Rootless & systemd Integration

### Running Podman Rootless

Rootless Podman runs entirely as a regular user — no root, no daemon, smaller attack surface.

```bash
# Run as regular user (no sudo needed)
podman run -d -p 8080:80 --name webserver nginx

# Check that no root is involved
podman info | grep -i rootless     # Should say "rootless: true"
id                                  # You are NOT root

# Rootless networking (uses slirp4netns or pasta)
podman network ls
```

### Podman Pods

Podman supports the concept of **pods** (like Kubernetes pods) — multiple containers sharing the same network namespace.

```bash
# Create a pod
podman pod create --name myapp -p 8080:80

# Add containers to the pod
podman run -d --pod myapp --name web nginx
podman run -d --pod myapp --name app myapp:1.0

# List pods
podman pod ls

# Stop/start entire pod
podman pod stop myapp
podman pod start myapp
```

### systemd Integration — Run Containers as Services

Podman can generate systemd unit files so containers start automatically on boot.

```bash
# Run your container first
podman run -d --name webserver --restart always nginx

# Generate a systemd unit file
podman generate systemd --name webserver --files --new

# This creates: container-webserver.service
# Install and enable the service
mkdir -p ~/.config/systemd/user/
cp container-webserver.service ~/.config/systemd/user/

systemctl --user daemon-reload
systemctl --user enable --now container-webserver.service
systemctl --user status container-webserver.service

# Enable linger so services start at boot without login
loginctl enable-linger $USER
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Container Security

### Key Security Principles

1. **Never run as root inside a container** — use a non-root USER in your Dockerfile
2. **Use minimal base images** — Alpine, Distroless, or scratch
3. **Scan images for vulnerabilities** before deploying
4. **Don't store secrets in images** — use environment variables, secrets managers
5. **Read-only filesystems** where possible
6. **Drop Linux capabilities** to only what's needed

```dockerfile
# Security-hardened Dockerfile
FROM python:3.12-slim

# Don't run as root
RUN useradd -r -u 1001 -s /bin/false appuser

WORKDIR /app
COPY --chown=appuser:appuser . .
RUN pip install --no-cache-dir -r requirements.txt

USER appuser
EXPOSE 8080
CMD ["python", "app.py"]
```

```bash
# Run with security options
docker run -d \
  --read-only \                          # Read-only filesystem
  --tmpfs /tmp \                         # Allow writes only to /tmp
  --cap-drop ALL \                       # Drop all capabilities
  --cap-add NET_BIND_SERVICE \           # Add back only what's needed
  --security-opt no-new-privileges \     # Prevent privilege escalation
  myapp:1.0

podman run -d \
  --read-only \
  --cap-drop ALL \
  --security-opt no-new-privileges \
  myapp:1.0
```

### Image Scanning with Trivy

```bash
# Install Trivy
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh

# Scan an image
trivy image nginx:1.25
trivy image --severity HIGH,CRITICAL myapp:1.0
trivy image --exit-code 1 --severity CRITICAL myapp:1.0  # Fail on critical CVEs
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Image Optimization Best Practices

```dockerfile
# BAD: Each RUN creates a new layer
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y nginx

# GOOD: Combine into one layer
RUN apt-get update && apt-get install -y \
    curl \
    nginx \
    && rm -rf /var/lib/apt/lists/*

# GOOD: Copy dependency files BEFORE source code (caching)
COPY package.json package-lock.json ./
RUN npm install
COPY . .    # Source code changes don't bust the npm install cache

# Use .dockerignore to exclude files from build context
# .dockerignore:
# .git
# node_modules
# *.log
# .env
```

### Choosing the Right Base Image

| Base Image | Size | Use Case |
|---|---|---|
| `ubuntu:24.04` | ~78 MB | General purpose, lots of tools available |
| `debian:bookworm-slim` | ~75 MB | Smaller Debian |
| `alpine:3.19` | ~5 MB | Minimal, for compiled apps |
| `python:3.12-slim` | ~45 MB | Python apps (slim variant) |
| `python:3.12-alpine` | ~20 MB | Python apps (smallest) |
| `gcr.io/distroless/python3` | ~15 MB | No shell, minimal attack surface |

[↑ Back to TOC](#table-of-contents)

---

## Tools & Commands Reference

| Command | Docker | Podman |
|---|---|---|
| Run container | `docker run` | `podman run` |
| List running | `docker ps` | `podman ps` |
| List images | `docker images` | `podman images` |
| Build image | `docker build` | `podman build` |
| Push image | `docker push` | `podman push` |
| Logs | `docker logs -f` | `podman logs -f` |
| Shell access | `docker exec -it` | `podman exec -it` |
| Inspect | `docker inspect` | `podman inspect` |
| Remove container | `docker rm -f` | `podman rm -f` |
| Remove image | `docker rmi` | `podman rmi` |
| Network create | `docker network create` | `podman network create` |
| Volume create | `docker volume create` | `podman volume create` |
| Compose up | `docker compose up -d` | `podman-compose up -d` |
| Login to registry | `docker login` | `podman login` |
| Generate systemd | N/A | `podman generate systemd` |
| Create pod | N/A | `podman pod create` |

[↑ Back to TOC](#table-of-contents)

---

## Hands-On Labs

### Lab 5.1 — Your First Container (Both Runtimes)

1. Pull and run `nginx` with Docker: `docker run -d -p 8080:80 --name nginx-docker nginx`
2. Do the same with Podman: `podman run -d -p 8081:80 --name nginx-podman nginx`
3. Test both: `curl http://localhost:8080` and `curl http://localhost:8081`
4. Inspect the running containers with both `docker ps` and `podman ps`
5. Stop and remove both containers

### Lab 5.2 — Build a Custom Image

1. Create a simple `index.html` file
2. Write a `Dockerfile` that uses `nginx:alpine` and copies your `index.html` into `/usr/share/nginx/html/`
3. Build the image: `docker build -t my-nginx:1.0 .`
4. Run it: `docker run -d -p 8080:80 my-nginx:1.0`
5. Test it: `curl http://localhost:8080`
6. Repeat steps 3–5 with Podman using `podman build` and `podman run`

### Lab 5.3 — Multi-Container App with Compose

1. Create a `docker-compose.yml` with a web service (nginx) and a Redis service
2. Start the stack: `docker compose up -d`
3. Verify both services are running: `docker compose ps`
4. View logs: `docker compose logs -f`
5. Run the same with `podman-compose`

### Lab 5.4 — Volumes & Persistence

1. Create a named volume: `docker volume create appdata`
2. Run a container with the volume: `docker run -it -v appdata:/data ubuntu:24.04 bash`
3. Create a file inside: `echo "persistent" > /data/test.txt`
4. Exit and remove the container
5. Run a new container with the same volume and confirm the file still exists

### Lab 5.5 — Podman systemd Service

1. Run an nginx container with Podman
2. Generate a systemd unit file: `podman generate systemd --name nginx-podman --files`
3. Install and enable the service
4. Reboot (or simulate reboot) and confirm the container starts automatically

### Lab 5.6 — Image Security Scan

1. Install Trivy
2. Scan `nginx:latest`: `trivy image nginx:latest`
3. Note the CVE counts by severity
4. Compare against `nginx:alpine`
5. Scan one of your custom-built images

[↑ Back to TOC](#table-of-contents)

---

## Further Reading

- [Docker Official Documentation](https://docs.docker.com/)
- [Podman Official Documentation](https://podman.io/docs)
- [OCI Image Specification](https://github.com/opencontainers/image-spec)
- [Trivy — Container Vulnerability Scanner](https://trivy.dev/)
- [Best Practices for Writing Dockerfiles](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [Rootless Containers](https://rootlesscontaine.rs/)
- [Glossary: Container](./glossary.md#c), [Image](./glossary.md#i), [Podman](./glossary.md#p), [Volume](./glossary.md#v)
- **Certification**: CKAD (Certified Kubernetes Application Developer) — containers are a prerequisite

[↑ Back to TOC](#table-of-contents)

---

*© 2026 UncleJS — Licensed under [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/)*
