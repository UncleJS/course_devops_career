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
- [Intermediate: Scaling Containers & Load Balancing](#intermediate-scaling-containers--load-balancing)
- [Intermediate: Container Registries](#intermediate-container-registries)
- [Intermediate: Podman Rootless & systemd Integration](#intermediate-podman-rootless--systemd-integration)
- [Intermediate: Container Security](#intermediate-container-security)
- [Intermediate: Image Optimization Best Practices](#intermediate-image-optimization-best-practices)
- [Advanced: Podman in Production](#advanced-podman-in-production)
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
- Explain Podman's rootless security model and user namespace remapping
- Run Podman in rootless mode and manage containers as unprivileged services
- Integrate Podman containers with systemd using both `podman generate systemd` and Quadlet unit files
- Write `.container`, `.network`, and `.pod` Quadlet unit files for production deployments
- Use `skopeo` to copy and inspect images across registries without pulling
- Use `buildah` for low-level OCI image construction
- Apply container security best practices including capability dropping and image scanning
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
| systemd integration | Limited | Native (Quadlet unit files) |
| CLI compatibility | `docker` | `podman` (drop-in replacement) |
| Compose support | `docker compose` | `podman-compose` / `podman compose` |
| Image format | OCI-compatible | OCI-compatible |
| Pod support | No | Yes (like Kubernetes pods) |
| Quadlet support | No | Yes (`.container` / `.network` / `.pod` units) |
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

## Intermediate: Scaling Containers & Load Balancing

Running a single container is fine for development. In production, you need multiple instances for availability and throughput — and a load balancer in front of them.

### Scaling with Docker Compose

Docker Compose can run multiple replicas of a service using `--scale`. Traffic distribution requires removing fixed host port mappings and placing a load balancer in front.

**Scalable `compose.yaml` (no fixed host port on the app):**

```yaml
version: "3.8"

services:
  web:
    image: nginx:alpine
    # No "ports:" here — the LB handles external traffic
    networks:
      - app-net

  lb:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx-lb.conf:/etc/nginx/nginx.conf:ro
    networks:
      - app-net
    depends_on:
      - web

networks:
  app-net:
```

**`nginx-lb.conf` — Nginx as the load balancer:**

```nginx
events { worker_connections 1024; }

http {
    upstream web_backends {
        least_conn;
        server web_1:80;    # Docker Compose names: <service>_<replica-number>
        server web_2:80;
        server web_3:80;
    }

    server {
        listen 80;
        location / {
            proxy_pass http://web_backends;
            proxy_set_header Host              $host;
            proxy_set_header X-Real-IP         $remote_addr;
            proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        }
    }
}
```

**Scale up and down:**

```bash
# Start with 3 replicas of the web service
docker compose up -d --scale web=3

# Scale up to 5 (live — no downtime)
docker compose up -d --scale web=5

# Scale down to 2
docker compose up -d --scale web=2

# Check how many replicas are running
docker compose ps
```

> **Note on naming**: Docker Compose names scaled containers `<project>-<service>-<n>` (e.g., `myapp-web-1`, `myapp-web-2`). Containers can reach each other using the service name — Docker's internal DNS resolves `web` to all containers in the service via round-robin.

### Traefik as a Compose-Native Load Balancer

Traefik is the cleanest way to load-balance Docker Compose services. It reads container labels — no manual upstream config needed. When you scale a service, Traefik **automatically detects the new replicas** and adds them to the pool.

**`compose.yaml` with Traefik auto-discovery:**

```yaml
version: "3.8"

networks:
  traefik-net:

services:
  traefik:
    image: traefik:v3
    command:
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--api.dashboard=true"
      - "--api.insecure=true"   # Dashboard on :8080 (dev only — protect in prod)
    ports:
      - "80:80"
      - "8080:8080"             # Dashboard
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - traefik-net

  web:
    image: traefik/whoami   # Simple test image — prints container info
    networks:
      - traefik-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.web.rule=Host(`app.localhost`)"
      - "traefik.http.routers.web.entrypoints=web"
      - "traefik.http.services.web.loadbalancer.server.port=80"
      # Round-robin across all replicas (default)
      - "traefik.http.services.web.loadbalancer.sticky.cookie=false"
```

```bash
# Start Traefik + 1 web instance
docker compose up -d

# Scale to 5 replicas — Traefik auto-discovers all of them
docker compose up -d --scale web=5

# Verify round-robin: each request should show a different container hostname
for i in $(seq 1 10); do
    curl -s -H "Host: app.localhost" http://localhost/ | grep Hostname
done

# Open Traefik dashboard to see the service with 5 backends
open http://localhost:8080/dashboard/
```

**Load balancing behavior**: By default Traefik uses round-robin across all healthy replicas. When you scale down, Traefik immediately stops routing to removed containers. When a new replica starts, it joins the pool after passing Traefik's health checks.

### Health Checks for Scaled Services

When scaling, health checks prevent traffic from reaching containers that haven't finished starting:

```yaml
services:
  web:
    image: myapp:latest
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 20s    # Grace period — no checks during app startup
    labels:
      - "traefik.enable=true"
      # Traefik respects Docker health status — only routes to healthy containers
      - "traefik.http.services.web.loadbalancer.healthcheck.path=/health"
      - "traefik.http.services.web.loadbalancer.healthcheck.interval=10s"
```

### Scaling in Production: Podman + Systemd

For rootless Podman in production (without Kubernetes), use systemd template units to run multiple instances:

```ini
# /etc/systemd/user/webapp@.service  (note the @ — this is a template unit)
[Unit]
Description=Web App Instance %i
After=network-online.target

[Service]
ExecStart=podman run --rm \
    --name webapp-%i \
    --network=webapp-net \
    --env-file=/etc/webapp/webapp.env \
    -p 0:3000 \
    ghcr.io/myorg/webapp:latest
ExecStop=podman stop webapp-%i
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=default.target
```

```bash
# Enable and start 3 instances
systemctl --user enable webapp@1 webapp@2 webapp@3
systemctl --user start webapp@1 webapp@2 webapp@3

# Check all instances
systemctl --user status 'webapp@*'

# Scale up: add a 4th
systemctl --user enable --now webapp@4
```

### Choosing Your Container LB Strategy

| Approach | Use When |
|---|---|
| **Nginx upstream block** | Simple, predictable — great when you know replica count ahead of time |
| **Traefik with Docker labels** | Dynamic scaling — Traefik auto-discovers new replicas without config changes |
| **HAProxy** | Maximum control, statistics, session persistence, connection draining |
| **Kubernetes Services** | You've outgrown Compose and need orchestration |

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

### The Rootless Security Model

Rootless Podman runs entirely as a regular user — no root, no daemon, dramatically smaller attack surface. Understanding *why* this is secure requires a look at Linux user namespaces.

#### User Namespace Remapping

When you run `podman run` as a non-root user, Podman creates a **user namespace** for the container. Inside that namespace, the process sees itself as UID 0 (root). On the host, it maps to your unprivileged UID.

```
Host                         Container (user namespace)
─────────────────────────    ──────────────────────────
UID 1000 (alice)      ──►    UID 0 (root inside container)
UID 1001              ──►    UID 1
UID 1002              ──►    UID 2
...                          ...
```

The mapping is stored in `/proc/self/uid_map`. The kernel enforces that even if a container process calls `setuid(0)`, it has no privileges outside its namespace.

```bash
# Confirm rootless is active
podman info | grep -i rootless     # rootless: true

# See the UID map for a running container
podman inspect <container> --format '{{.HostConfig.UsernsMode}}'

# View the actual mapping at the kernel level
cat /proc/self/uid_map
# Output: 0  1000  1   (container UID 0 = host UID 1000, 1 mapping)
```

#### Rootless vs Root-daemon Comparison

| Aspect | Docker (root daemon) | Podman (rootless) |
|---|---|---|
| Daemon | `dockerd` runs as root | No daemon at all |
| Socket | `/var/run/docker.sock` (world-writable risk) | Per-user: `$XDG_RUNTIME_DIR/podman/podman.sock` |
| Privilege escalation risk | Compromised socket = root on host | No socket, no daemon to compromise |
| Networking | Full bridge networking | `slirp4netns` or `pasta` (userspace networking) |
| Port < 1024 | Can bind directly | Requires port forwarding workaround |
| Storage | `/var/lib/docker` | `~/.local/share/containers` |

#### Rootless Networking

Rootless containers use userspace network stacks because creating real network interfaces requires `CAP_NET_ADMIN`.

- **`slirp4netns`** — legacy userspace TCP/IP stack; slower but widely supported
- **`pasta`** — modern replacement (Podman 4.4+); faster, better IPv6 support

```bash
# Check which network backend is in use
podman info | grep networkBackend

# Rootless containers can expose ports >= 1024 natively
podman run -d -p 8080:80 --name webserver nginx

# To bind port 80 (privileged) as rootless — use sysctl or a higher port
# Option 1: Map host 80 → container 80 via firewall redirect (recommended)
sudo sysctl net.ipv4.ip_unprivileged_port_start=80

# Option 2: Use a host port >= 1024 and put a reverse proxy in front
podman run -d -p 8080:80 nginx
# Then configure Nginx/Traefik on port 80 to forward to localhost:8080
```

---

### Podman Desktop

**Podman Desktop** is the official GUI for managing rootless containers locally — available for Linux, macOS, and Windows. It's particularly useful for developers who want Docker Desktop parity without the licensing concerns.

**Key capabilities:**
- Start/stop/inspect containers, pods, images, and volumes
- View real-time logs and resource usage
- Port-forward into running containers
- Manage Compose stacks (using `podman compose`)
- Connect to remote Podman sockets (including WSL2 or SSH remotes)
- Kubernetes integration — deploy to local `kind`/`minikube` clusters

```bash
# Install on Linux (Flatpak)
flatpak install flathub io.podman_desktop.PodmanDesktop

# Or download AppImage from podman-desktop.io
```

> **DevOps tip**: Podman Desktop's "Extensions" system lets you add Kubernetes, Red Hat OpenShift, and Compose plugins. It's a compelling replacement for Docker Desktop on developer workstations in RHEL/Fedora shops.

---

### Running Podman Rootless

```bash
# Run as regular user (no sudo needed)
podman run -d -p 8080:80 --name webserver nginx

# Check that no root is involved
podman info | grep -i rootless     # Should say "rootless: true"
id                                  # You are NOT root

# Rootless networking
podman network ls

# User-scoped storage location
ls ~/.local/share/containers/storage/
```

---

### Podman Pods

Podman supports the concept of **pods** (like Kubernetes pods) — multiple containers sharing the same network namespace. This is unique to Podman and has no Docker equivalent.

```
Pod: myapp
├── infra container (pause)   ← holds the shared network namespace
├── web (nginx)               ← shares pod's IP and ports
└── app (your service)        ← shares pod's IP and ports
```

The **infra container** (`k8s.gcr.io/pause` or `localhost/podman-pause`) holds the network namespace open. Other containers in the pod join it. If a container restarts, the network namespace persists — just like Kubernetes.

```bash
# Create a pod with a published port
podman pod create --name myapp -p 8080:80

# Add containers to the pod (they share the pod's network)
podman run -d --pod myapp --name web nginx
podman run -d --pod myapp --name app myapp:1.0

# List pods and their containers
podman pod ls
podman pod ps

# Resource stats for the whole pod
podman pod stats myapp

# Stop/start/remove entire pod
podman pod stop myapp
podman pod start myapp
podman pod rm -f myapp   # removes pod and all its containers

# Generate Kubernetes YAML from a running pod (bridge to Module 06!)
podman kube generate myapp -f myapp-pod.yaml
```

**Sidecar pattern example** — app container + log shipper sharing the same network:

```bash
podman pod create --name logsidecar
podman run -d --pod logsidecar --name app myapp:1.0
podman run -d --pod logsidecar --name fluentbit \
  -v /var/log/app:/var/log/app:ro \
  fluent/fluent-bit:latest
# Both containers reach each other via localhost inside the pod
```

---

### systemd Integration — Two Approaches

Podman offers two ways to run containers as systemd services:

| Approach | Status | Best for |
|---|---|---|
| `podman generate systemd` | Deprecated (Podman v4.4+) | Legacy systems, quick one-offs |
| **Quadlet** (`.container` unit files) | **Recommended** | Production, declarative, RHEL 9+ |

---

#### Approach A: `podman generate systemd` (Legacy)

This approach generates a `.service` file from a *running* container. Still useful for older systems or quick setups.

```bash
# 1. Run your container first
podman run -d --name webserver --restart always nginx

# 2. Generate the systemd unit file
#    --new: creates a fresh container on start (rather than managing the existing one)
#    --files: write to disk rather than stdout
podman generate systemd --name webserver --files --new

# 3. Install the generated unit
mkdir -p ~/.config/systemd/user/
cp container-webserver.service ~/.config/systemd/user/

# 4. Enable and start
systemctl --user daemon-reload
systemctl --user enable --now container-webserver.service
systemctl --user status container-webserver.service

# 5. Enable linger so the service starts at boot without logging in
loginctl enable-linger $USER

# 6. View logs via the systemd journal
journalctl --user -u container-webserver.service -f
```

> ⚠️ `podman generate systemd` is deprecated as of Podman 4.4. Use Quadlet for new deployments.

---

#### Approach B: Quadlet (Modern — Recommended)

**Quadlet** is the production-grade, declarative way to run Podman containers as systemd services. You write a `.container` unit file and systemd's generator automatically creates the corresponding `.service` unit at `daemon-reload` time.

**File locations:**

| Scope | Directory |
|---|---|
| User (rootless) | `~/.config/containers/systemd/` |
| System (root) | `/etc/containers/systemd/` |

**Unit file types:**

| Extension | Purpose |
|---|---|
| `.container` | Define a single container as a service |
| `.network` | Define a Podman network (CNI/Netavark) |
| `.pod` | Define a Podman pod |
| `.volume` | Define a named Podman volume |
| `.image` | Pre-pull an image as a systemd dependency |

##### Example: Simple web server

```ini
# ~/.config/containers/systemd/webserver.container

[Unit]
Description=Nginx Web Server
After=network-online.target

[Container]
Image=docker.io/library/nginx:alpine
PublishPort=8080:80
Volume=%h/www:/usr/share/nginx/html:ro,z
Environment=NGINX_HOST=localhost
AutoUpdate=registry

[Service]
Restart=always
TimeoutStartSec=30

[Install]
WantedBy=default.target
```

```bash
# Reload systemd — Quadlet generator creates the .service unit automatically
systemctl --user daemon-reload

# The service name is derived from the filename (without .container)
systemctl --user start webserver
systemctl --user enable webserver
systemctl --user status webserver

# Logs
journalctl --user -u webserver -f
```

##### Example: Network + two containers

```ini
# ~/.config/containers/systemd/appnet.network
[Network]
Driver=bridge
Subnet=10.88.1.0/24
```

```ini
# ~/.config/containers/systemd/redis.container
[Unit]
Description=Redis Cache
After=network-online.target

[Container]
Image=docker.io/library/redis:7-alpine
Network=appnet.network
Volume=redis-data:/data:Z

[Service]
Restart=always
```

```ini
# ~/.config/containers/systemd/myapp.container
[Unit]
Description=My Application
After=redis.service

[Container]
Image=ghcr.io/uncleJS/myapp:1.0
Network=appnet.network
PublishPort=3000:3000
Environment=REDIS_URL=redis://redis:6379
HealthCmd=curl -f http://localhost:3000/health || exit 1
HealthInterval=30s
HealthRetries=3

[Service]
Restart=on-failure

[Install]
WantedBy=default.target
```

```bash
# Apply all three units at once
systemctl --user daemon-reload

# Start in dependency order
systemctl --user start myapp   # starts redis first (After=redis.service)

# Validate unit files before applying
systemd-analyze --user verify ~/.config/containers/systemd/myapp.container
```

##### Key Quadlet directives reference

| Directive | Description |
|---|---|
| `Image=` | Container image to use |
| `PublishPort=` | `hostPort:containerPort` |
| `Volume=` | Mount: `hostPath:containerPath:options` |
| `Network=` | Attach to a `.network` unit or existing network |
| `Environment=` | Set env var (`KEY=value`) |
| `EnvironmentFile=` | Load env vars from a file |
| `Secret=` | Mount a Podman secret by name |
| `HealthCmd=` | Health check command |
| `HealthInterval=` | Interval between health checks |
| `AutoUpdate=` | `registry` = auto-pull new image tags |
| `Pod=` | Attach to a `.pod` unit |
| `Label=` | Add container label |
| `Exec=` | Override container entrypoint command |

```bash
# Enable linger for rootless services to start at boot
loginctl enable-linger $USER
loginctl show-user $USER | grep Linger
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

## Advanced: Podman in Production

This section covers production-grade Podman patterns: OCI builds without Docker, CI/CD integration, image registry management with `skopeo`, secrets management, and a complete multi-container Quadlet stack.

---

### OCI Image Building Without Docker

Podman can build OCI-compliant images without the Docker daemon, making it ideal for rootless CI pipelines.

#### Containerfile vs Dockerfile

`Containerfile` is the OCI-standard name for what Docker calls a `Dockerfile`. The syntax is identical — Podman accepts both names automatically.

```bash
# Both work — Podman checks for Containerfile first, then Dockerfile
podman build -t myapp:1.0 .

# Explicitly specify the file
podman build -f Containerfile -t myapp:1.0 .
podman build -f Dockerfile -t myapp:1.0 .
```

#### Multi-Platform Builds

```bash
# Build for multiple CPU architectures
podman build \
  --platform linux/amd64,linux/arm64 \
  --manifest myapp:1.0 \
  .

# Push the multi-arch manifest to a registry
podman manifest push myapp:1.0 ghcr.io/uncleJs/myapp:1.0
```

#### buildah — Low-Level OCI Builder

`buildah` is the underlying OCI image build tool that Podman uses internally. It's useful when you need fine-grained control over image layers.

```bash
# Install
sudo dnf install -y buildah    # RHEL/Fedora
sudo apt install -y buildah    # Ubuntu

# Build from a Containerfile (same as podman build)
buildah bud -t myapp:1.0 .

# Script-based build (no Containerfile at all)
container=$(buildah from ubi9-minimal)
buildah run $container -- dnf install -y python3
buildah copy $container app.py /opt/app.py
buildah config --cmd "python3 /opt/app.py" $container
buildah commit $container myapp:1.0

# Push to registry
buildah push myapp:1.0 ghcr.io/uncleJs/myapp:1.0
```

> **buildah vs podman build**: Use `podman build` for everyday image builds. Use `buildah` when you need scripted, layer-by-layer control — for example, in pipeline stages that add security patches to a base image without a full rebuild.

---

### Podman in CI/CD

#### GitHub Actions — Rootless Podman

GitHub-hosted runners include Podman. Replace `docker` with `podman` — no additional setup needed.

```yaml
# .github/workflows/build.yml
name: Build & Push Image

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Log in to GHCR
        run: echo "${{ secrets.GITHUB_TOKEN }}" | podman login ghcr.io -u ${{ github.actor }} --password-stdin

      - name: Build image
        run: podman build -t ghcr.io/${{ github.repository }}:${{ github.sha }} .

      - name: Push image
        run: podman push ghcr.io/${{ github.repository }}:${{ github.sha }}

      - name: Scan for vulnerabilities
        run: |
          curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh
          trivy image --exit-code 1 --severity CRITICAL \
            ghcr.io/${{ github.repository }}:${{ github.sha }}
```

#### GitLab CI — Rootless Podman (No `--privileged`)

Traditional Docker-in-Docker requires `--privileged`. Podman avoids this entirely via its rootless model.

```yaml
# .gitlab-ci.yml
variables:
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

build:
  image: quay.io/podman/stable:latest
  script:
    - podman login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - podman build -t $IMAGE_TAG .
    - podman push $IMAGE_TAG
  # No 'privileged: true' needed!
```

#### Podman Socket — Docker API Drop-In

If your CI tools only speak the Docker API (e.g., older Compose tools, Testcontainers), expose Podman's socket as a Docker API replacement:

```bash
# Enable and start the Podman socket (user scope)
systemctl --user enable --now podman.socket

# The socket path
echo $XDG_RUNTIME_DIR/podman/podman.sock
# e.g. /run/user/1000/podman/podman.sock

# Point Docker CLI (or any Docker-compatible tool) at the Podman socket
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/podman/podman.sock
docker ps    # ← actually talking to Podman
docker compose up -d    # ← Compose running against Podman
```

---

### Image Registry Workflow with skopeo

`skopeo` is a command-line tool for working with remote container image registries — copying, inspecting, signing, and syncing — **without pulling images to local storage first**.

```bash
# Install
sudo dnf install -y skopeo    # RHEL/Fedora
sudo apt install -y skopeo    # Ubuntu

# Inspect a remote image without pulling it
skopeo inspect docker://docker.io/library/nginx:alpine
skopeo inspect docker://ghcr.io/uncleJs/myapp:1.0

# Copy image between registries (no local pull needed)
skopeo copy \
  docker://docker.io/library/nginx:alpine \
  docker://ghcr.io/uncleJs/nginx:alpine

# Copy to/from local OCI directory
skopeo copy docker://nginx:alpine oci:/tmp/nginx-oci

# Sync an entire repository (all tags)
skopeo sync \
  --src docker --dest docker \
  docker.io/library/nginx ghcr.io/uncleJs/mirror/

# Delete a remote tag
skopeo delete docker://ghcr.io/uncleJs/myapp:old-tag

# List available tags
skopeo list-tags docker://docker.io/library/nginx
```

#### Registry Configuration (`registries.conf`)

On RHEL/Fedora systems, Podman and `skopeo` use `/etc/containers/registries.conf` to configure search order, mirrors, and blocked registries:

```toml
# /etc/containers/registries.conf

# Default search registries (checked in order when image has no prefix)
unqualified-search-registries = ["registry.access.redhat.com", "docker.io"]

# Mirror a registry for air-gapped/pull-through caching
[[registry]]
prefix = "docker.io"
location = "docker.io"

  [[registry.mirror]]
  location = "mirror.internal.corp/docker-cache"

# Block a registry entirely
[[registry]]
prefix = "untrusted-registry.example.com"
blocked = true
```

```bash
# Per-user override (takes precedence over /etc)
mkdir -p ~/.config/containers/
# Edit ~/.config/containers/registries.conf

# Test which registry would be used
podman image search --list-tags nginx
```

---

### Secrets Management

Podman has a built-in secrets manager that stores sensitive data and mounts it into containers at runtime — keeping secrets out of image layers and environment variables.

```bash
# Create a secret from a literal value
echo "supersecretpassword" | podman secret create db-password -

# Create a secret from a file
podman secret create tls-cert /path/to/cert.pem

# List secrets (values are never shown)
podman secret ls

# Inspect a secret (metadata only, not the value)
podman secret inspect db-password

# Use a secret in a container (mounted as a file under /run/secrets/)
podman run -d \
  --secret db-password \
  --name myapp \
  myapp:1.0
# Secret is accessible at /run/secrets/db-password inside the container

# Custom mount path
podman run -d \
  --secret db-password,target=/etc/myapp/db.password \
  myapp:1.0

# Use in a Quadlet unit
# ~/.config/containers/systemd/myapp.container
# [Container]
# Secret=db-password,target=/run/secrets/db-password

# Remove a secret
podman secret rm db-password
```

---

### Full Production Quadlet Stack

A complete three-tier application deployed entirely with Quadlet unit files: Nginx frontend → Node.js API → Redis cache.

```
~/
└── .config/containers/systemd/
    ├── prod.network        ← shared bridge network
    ├── redis.container     ← Redis cache
    ├── api.container       ← Node.js API (depends on Redis)
    └── frontend.container  ← Nginx reverse proxy (depends on API)
```

#### Step 1 — Create the network unit

```ini
# ~/.config/containers/systemd/prod.network
[Network]
Driver=bridge
Subnet=10.89.0.0/24
Label=app=myapp
Label=env=production
```

#### Step 2 — Redis container unit

```ini
# ~/.config/containers/systemd/redis.container
[Unit]
Description=Redis Cache (myapp)
After=network-online.target

[Container]
Image=docker.io/library/redis:7-alpine
Network=prod.network
Volume=redis-data:/data:Z
Exec=redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}
EnvironmentFile=%h/.config/containers/systemd/myapp.env
HealthCmd=redis-cli -a ${REDIS_PASSWORD} ping
HealthInterval=15s
HealthRetries=3
Label=app=myapp
Label=tier=cache

[Service]
Restart=always
RestartSec=5s
```

#### Step 3 — API container unit

```ini
# ~/.config/containers/systemd/api.container
[Unit]
Description=Node.js API (myapp)
After=redis.service
Requires=redis.service

[Container]
Image=ghcr.io/uncleJs/myapp-api:1.0
Network=prod.network
PublishPort=127.0.0.1:3000:3000
EnvironmentFile=%h/.config/containers/systemd/myapp.env
Secret=db-password,target=/run/secrets/db-password
HealthCmd=curl -sf http://localhost:3000/health || exit 1
HealthInterval=20s
HealthRetries=3
AutoUpdate=registry
Label=app=myapp
Label=tier=api

[Service]
Restart=on-failure
RestartSec=10s

[Install]
WantedBy=default.target
```

#### Step 4 — Frontend container unit

```ini
# ~/.config/containers/systemd/frontend.container
[Unit]
Description=Nginx Frontend (myapp)
After=api.service
Requires=api.service

[Container]
Image=ghcr.io/uncleJs/myapp-frontend:1.0
Network=prod.network
PublishPort=8080:80
Volume=%h/conf/nginx.conf:/etc/nginx/nginx.conf:ro,Z
HealthCmd=curl -sf http://localhost/ || exit 1
HealthInterval=30s
AutoUpdate=registry
Label=app=myapp
Label=tier=frontend

[Service]
Restart=always

[Install]
WantedBy=default.target
```

#### Step 5 — Environment file

```bash
# ~/.config/containers/systemd/myapp.env
REDIS_PASSWORD=changeme_in_production
REDIS_URL=redis://:changeme_in_production@redis:6379
NODE_ENV=production
API_BASE=http://api:3000
```

```bash
# Secure the env file
chmod 600 ~/.config/containers/systemd/myapp.env
```

#### Step 6 — Deploy

```bash
# Validate all unit files
systemd-analyze --user verify \
  ~/.config/containers/systemd/redis.container \
  ~/.config/containers/systemd/api.container \
  ~/.config/containers/systemd/frontend.container

# Apply
systemctl --user daemon-reload

# Start (systemd resolves dependency order automatically)
systemctl --user start frontend

# Verify
systemctl --user status frontend api redis
podman ps
podman pod ps

# Enable for auto-start at boot
systemctl --user enable frontend
loginctl enable-linger $USER

# View aggregated logs
journalctl --user -u frontend -u api -u redis -f

# Check health status
podman healthcheck run api
```

#### Auto-update with `podman-auto-update`

When `AutoUpdate=registry` is set in a `.container` unit, Podman can automatically pull new image versions:

```bash
# Enable the auto-update timer (checks daily by default)
systemctl --user enable --now podman-auto-update.timer

# Manual update check
podman auto-update

# Dry run (shows what would be updated)
podman auto-update --dry-run
```

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
| Compose up | `docker compose up -d` | `podman compose up -d` / `podman-compose up -d` |
| Login to registry | `docker login` | `podman login` |
| Generate systemd (legacy) | N/A | `podman generate systemd --name <c> --files --new` |
| Quadlet unit dir (user) | N/A | `~/.config/containers/systemd/` |
| Create pod | N/A | `podman pod create` |
| Pod list | N/A | `podman pod ls` |
| Pod stats | N/A | `podman pod stats` |
| Generate K8s YAML | N/A | `podman kube generate` |
| Run K8s YAML | N/A | `podman kube play` |
| Secret create | N/A | `podman secret create <name> -` |
| Secret list | N/A | `podman secret ls` |
| Auto-update check | N/A | `podman auto-update` |
| Image inspect (remote) | N/A | `skopeo inspect docker://<image>` |
| Image copy (registry→registry) | N/A | `skopeo copy docker://<src> docker://<dst>` |
| Image sync (repo mirror) | N/A | `skopeo sync --src docker --dest docker <repo> <dst>` |
| List remote tags | N/A | `skopeo list-tags docker://<image>` |
| Low-level OCI build | N/A | `buildah bud -t <tag> .` |
| Enable linger | N/A | `loginctl enable-linger $USER` |

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

### Lab 5.5 — Podman systemd Service (Both Approaches)

Practice both the legacy and modern methods for running Podman containers as systemd services.

**Path A — Legacy: `podman generate systemd`**

1. Run an nginx container: `podman run -d --name nginx-legacy nginx:alpine`
2. Generate the unit file: `podman generate systemd --name nginx-legacy --files --new`
3. Install it: `mkdir -p ~/.config/systemd/user/ && cp container-nginx-legacy.service ~/.config/systemd/user/`
4. Enable and start: `systemctl --user daemon-reload && systemctl --user enable --now container-nginx-legacy.service`
5. Verify: `systemctl --user status container-nginx-legacy.service`
6. View logs: `journalctl --user -u container-nginx-legacy.service -n 20`
7. Stop and disable the service, then remove the unit file

**Path B — Modern: Quadlet**

1. Create the Quadlet unit file:
   ```bash
   mkdir -p ~/.config/containers/systemd/
   ```
   Write `~/.config/containers/systemd/nginx-quadlet.container`:
   ```ini
   [Unit]
   Description=Nginx via Quadlet

   [Container]
   Image=docker.io/library/nginx:alpine
   PublishPort=8081:80

   [Service]
   Restart=always

   [Install]
   WantedBy=default.target
   ```
2. Validate the unit: `systemd-analyze --user verify ~/.config/containers/systemd/nginx-quadlet.container`
3. Apply: `systemctl --user daemon-reload`
4. Start: `systemctl --user start nginx-quadlet`
5. Test: `curl http://localhost:8081`
6. View logs: `journalctl --user -u nginx-quadlet -f`
7. Enable for boot: `systemctl --user enable nginx-quadlet && loginctl enable-linger $USER`
8. Confirm linger is active: `loginctl show-user $USER | grep Linger`

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
- [Podman Desktop](https://podman-desktop.io/)
- [Quadlet — Podman systemd integration](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html)
- [skopeo — Image Registry Tool](https://github.com/containers/skopeo)
- [buildah — OCI Image Builder](https://buildah.io/)
- [OCI Image Specification](https://github.com/opencontainers/image-spec)
- [Trivy — Container Vulnerability Scanner](https://trivy.dev/)
- [Best Practices for Writing Dockerfiles](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [Rootless Containers](https://rootlesscontaine.rs/)
- [pasta — Fast Rootless Networking](https://passt.top/)
- [Glossary: Container](./glossary.md#c), [Image](./glossary.md#i), [Podman](./glossary.md#p), [Volume](./glossary.md#v)
- **Certification**: CKAD (Certified Kubernetes Application Developer) — containers are a prerequisite

[↑ Back to TOC](#table-of-contents)

---

*© 2026 UncleJS — Licensed under [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/)*
