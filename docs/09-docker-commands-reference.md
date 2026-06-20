# 09 — Docker Commands Reference

> Every Docker command you'll ever need — with examples and explanations

---

## Table of Contents

1. [Image Commands](#image-commands)
2. [Container Commands](#container-commands)
3. [Volume Commands](#volume-commands)
4. [Network Commands](#network-commands)
5. [System Commands](#system-commands)
6. [Build Commands](#build-commands)
7. [Compose Commands](#compose-commands)
8. [Registry Commands](#registry-commands)
9. [Context & Swarm Commands](#context--swarm-commands)

---

## Image Commands

### docker images / docker image ls

List all images on the local system.

```bash
# List all images
docker images
docker image ls

# List with filter
docker images -f dangling=true            # Untagged images
docker images -f before=nginx:latest      # Images built before nginx
docker images -f since=nginx:latest       # Images built since nginx
docker images -f label=env=production     # Images with specific label

# Custom format
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"
docker images --format "{{.ID}}: {{.Repository}}:{{.Tag}}"

# Show digests (SHA256 hashes)
docker images --digests

# Quiet mode (just IDs)
docker images -q

# Last N images
docker images -n 5
```

### docker pull

Pull an image (and its layers) from a registry.

```bash
# Pull latest
docker pull nginx

# Pull specific tag
docker pull nginx:1.25-alpine

# Pull from private registry
docker pull registry.example.com:5000/myapp:1.0.0

# Pull all tags
docker pull --all-tags nginx

# Pull for different platform
docker pull --platform linux/arm64 nginx

# Quiet mode
docker pull -q nginx
```

**Why use it:** Pull images before running to avoid delays at container start.

### docker push

Push an image to a registry.

```bash
# Push to Docker Hub
docker push myuser/myapp:latest

# Push to private registry
docker push registry.example.com/myapp:1.0.0

# Push all tags
docker push --all-tags myuser/myapp

# Push specific platform
docker push --platform linux/amd64 myuser/myapp:latest
```

**Why use it:** Distribute images to other machines or CI/CD systems.

### docker rmi / docker image rm

Remove one or more images.

```bash
# Remove by name:tag
docker rmi nginx:latest

# Remove by image ID
docker rmi abc123def456

# Remove multiple
docker rmi nginx:latest node:20-alpine

# Force remove (even if used by containers)
docker rmi -f nginx:latest

# Remove all unused
docker image prune
docker image prune -a          # Also remove dangling images

# Remove dangling (untagged) images only
docker image prune -f

# Remove images with filter
docker image prune -a --filter until=24h   # Older than 24h
docker image prune -a --filter label=env=staging
```

**Why use it:** Free up disk space from unused images.

### docker tag

Create a tag for an image.

```bash
# Tag for versioning
docker tag myapp:latest myapp:1.0.0

# Tag for registry
docker tag myapp:latest registry.example.com/myapp:1.0.0

# Tag with multiple tags
docker tag myapp:latest myapp:stable myapp:1.0.0

# Retag for different registry
docker tag nginx:latest myregistry/nginx:custom
```

**Why use it:** Images are identified by name:tag. Tagging is how you version and distribute images.

### docker build

Build an image from a Dockerfile.

```bash
# Basic build
docker build -t myapp:latest .

# Build with Dockerfile in different location
docker build -f ./deploy/Dockerfile.prod -t myapp:prod .

# Build with build args
docker build --build-arg VERSION=1.0.0 --build-arg NODE_ENV=production -t myapp .

# Build for multiple platforms
docker build --platform linux/amd64,linux/arm64 -t myapp:latest .

# Build without cache
docker build --no-cache -t myapp:latest .

# Build with specific tag only
docker build -t myapp:1.0.0 .
```

**Why use it:** Convert your Dockerfile + source code into a deployable image.

### docker history

Show the history of an image (all layers and their sizes).

```bash
# Show layers
docker history nginx:latest

# Show with details
docker history --no-trunc nginx:latest

# Show with format
docker history --format "table {{.ID}}\t{{.CreatedBy}}\t{{.Size}}" nginx:latest
```

**Why use it:** Debug why an image is large, see what each layer contains.

### docker save / load

Save/load images as tar files (for offline transfer).

```bash
# Save image to tar
docker save myapp:latest -o myapp.tar

# Save multiple
docker save myapp:latest nginx:latest -o images.tar

# Compress
docker save myapp:latest | gzip > myapp.tar.gz

# Load from tar
docker load -i myapp.tar

# Load from compressed
gunzip -c myapp.tar.gz | docker load
```

**Why use it:** Transfer images to air-gapped systems or offline environments.

### docker inspect (image)

View detailed metadata about an image.

```bash
# Full JSON
docker inspect nginx:latest

# Get specific field
docker inspect --format '{{.Config.ExposedPorts}}' nginx:latest
docker inspect --format '{{.Config.Env}}' nginx:latest
docker inspect --format '{{.Architecture}}' nginx:latest
docker inspect --format '{{.RootFS.Layers}}' nginx:latest
```

**Why use it:** Understand image configuration, env vars, ports, layers.

---

## Container Commands

### docker run

Create and start a container from an image.

```bash
# Basic
docker run nginx

# Detached (background)
docker run -d nginx

# With port mapping
docker run -d -p 8080:80 nginx

# With name
docker run -d --name mynginx nginx

# With environment variables
docker run -d -e NODE_ENV=production -e PORT=3000 myapp

# With volume mount
docker run -d -v mydata:/data postgres

# With resource limits
docker run -d --cpus=2 --memory=512m app

# Interactive + TTY (for shell access)
docker run -it ubuntu bash

# Auto-remove on exit
docker run --rm -it ubuntu bash

# Restart policy
docker run -d --restart always nginx

# Network
docker run -d --network=my-network --name app app

# Multiple flags
docker run -d \
  --name myapp \
  --restart unless-stopped \
  -p 3000:3000 \
  -v app-data:/app/data \
  -e NODE_ENV=production \
  --cpus=2 \
  --memory=512m \
  --network=my-network \
  --health-cmd="curl -f http://localhost:3000/health" \
  --health-interval=30s \
  myapp:latest
```

**Why use it:** The primary command for creating containers from images.

### docker ps

List containers.

```bash
# Running containers
docker ps

# All containers (including stopped)
docker ps -a

# Last N
docker ps -n 5

# Filter by status
docker ps -f status=running
docker ps -f status=exited
docker ps -f status=created

# Filter by name
docker ps -f name=myapp

# Filter by image
docker ps -f ancestor=nginx

# Filter by label
docker ps -f label=env=production

# Filter by health
docker ps -f health=healthy
docker ps -f health=unhealthy

# Quiet (just IDs)
docker ps -q

# With size
docker ps -s

# Custom format
docker ps --format "table {{.ID}}\t{{.Names}}\t{{.Status}}\t{{.Ports}}"
docker ps --format "{{.ID}}: {{.Names}} ({{.Image}})"
```

**Why use it:** Check what containers are running and their status.

### docker stop / start / restart

Manage container lifecycle.

```bash
# Stop (SIGTERM, then SIGKILL after timeout)
docker stop mycontainer
docker stop mycontainer1 mycontainer2    # Multiple
docker stop $(docker ps -q)              # All running

# Stop with custom timeout
docker stop -t 30 mycontainer

# Start stopped container
docker start mycontainer
docker start -a mycontainer              # Attach output

# Restart
docker restart mycontainer
docker restart -t 30 mycontainer
```

**Why use it:** Control container execution.

### docker kill

Send a signal to a container (default: SIGKILL).

```bash
# Kill immediately
docker kill mycontainer

# Send custom signal
docker kill -s SIGQUIT mycontainer
docker kill --signal=USR1 mycontainer
```

**Why use it:** Force-stop unresponsive containers that don't respond to `docker stop`.

### docker rm

Remove one or more containers.

```bash
# Remove stopped container
docker rm mycontainer

# Remove running container (force)
docker rm -f mycontainer

# Remove multiple
docker rm container1 container2

# Remove all stopped
docker rm $(docker ps -a -q)
docker container prune

# Remove with volumes
docker rm -v mycontainer

# Remove with filter
docker container prune --filter until=24h
```

**Why use it:** Clean up stopped containers to free disk space.

### docker exec

Run a command in a running container.

```bash
# Interactive shell
docker exec -it mycontainer bash
docker exec -it mycontainer sh
docker exec -it mycontainer /bin/sh

# Run a single command
docker exec mycontainer ls -la /app
docker exec mycontainer cat /etc/hosts
docker exec mycontainer whoami

# Run as different user
docker exec -u node mycontainer whoami

# Run in specific directory
docker exec -w /app mycontainer pwd

# With environment variable
docker exec -e DEBUG=true mycontainer env

# Detached exec
docker exec -d mycontainer touch /tmp/healthcheck

# Non-interactive (CI/CD)
docker exec -T mycontainer npm test
```

**Why use it:** Debug running containers, run maintenance commands.

### docker logs

Fetch logs from a container.

```bash
# All logs
docker logs mycontainer

# Follow (like tail -f)
docker logs -f mycontainer

# Last N lines
docker logs --tail 100 mycontainer

# Since time
docker logs --since 2024-01-01T00:00:00 mycontainer
docker logs --since 5m
docker logs --since 1h30m

# Until time
docker logs --until 2024-01-02T00:00:00 mycontainer

# With timestamps
docker logs -t mycontainer

# Combine flags
docker logs -f --tail 50 -t mycontainer

# Show only from last 100 lines since 10 minutes ago
docker logs --since 10m --tail 100 mycontainer

# JSON output for processing
docker logs mycontainer 2>&1 | jq '.message'
```

**Why use it:** Debug issues, check application output.

### docker logs — Log Drivers

```bash
# json-file (default) — structured JSON
docker logs mycontainer

# journald — systemd journal
docker run --log-driver journald app
journalctl -u docker.service --since "5m ago"

# syslog — remote syslog
docker run --log-driver syslog --log-opt syslog-address=tcp://logs.example.com:514 app

# fluentd
docker run --log-driver fluentd --log-opt fluentd-address=localhost:24224 app

# awslogs — CloudWatch
docker run --log-driver awslogs --log-opt awslogs-group=myapp --log-opt awslogs-stream=myapp app

# local — fast, efficient for dev
docker run --log-driver local app

# none — no logs
docker run --log-driver none app
```

**Why use it:** Different environments need different log delivery mechanisms.

### docker inspect (container)

```bash
# Full JSON
docker inspect mycontainer

# Get IP address
docker inspect --format '{{.NetworkSettings.IPAddress}}' mycontainer

# Get port mapping
docker inspect --format '{{json .NetworkSettings.Ports}}' mycontainer | jq

# Get status
docker inspect --format '{{.State.Status}}' mycontainer

# Get health status
docker inspect --format '{{.State.Health.Status}}' mycontainer

# Get mount info
docker inspect --format '{{json .Mounts}}' mycontainer | jq

# Get log path
docker inspect --format '{{.LogPath}}' mycontainer
```

**Why use it:** Debug configuration, network, volumes from running containers.

### docker cp

Copy files between host and container.

```bash
# Copy from container to host
docker cp mycontainer:/app/logs/app.log ./logs/
docker cp mycontainer:/etc/nginx/nginx.conf ./config/

# Copy from host to container
docker cp ./config.json mycontainer:/app/config.json
docker cp ./init.sql postgres:/docker-entrypoint-initdb.d/

# Copy between containers
docker cp mycontainer1:/data/file.txt mycontainer2:/backup/
```

**Why use it:** Transfer files without mounting volumes.

### docker stats

Live resource usage statistics.

```bash
# All running containers
docker stats

# Specific containers
docker stats myapp mydb

# No-stream (single snapshot)
docker stats --no-stream

# Custom format
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}"

# JSON for processing
docker stats --no-stream --format "{{json .}}"
```

**Why use it:** Monitor CPU, memory, network, and disk I/O in real-time.

### docker top

Show processes running inside a container.

```bash
# All processes
docker top mycontainer

# Custom format
docker top mycontainer -eo pid,ppid,cmd,%cpu,%mem
```

**Why use it:** Check what processes are running inside the container.

### docker port

List port mappings for a container.

```bash
# All ports
docker port mycontainer
# 80/tcp -> 0.0.0.0:8080

# Specific port
docker port mycontainer 80
# 0.0.0.0:8080
```

**Why use it:** Find out which host port a container's port maps to.

### docker rename

```bash
docker rename old-name new-name
```

**Why use it:** Rename containers without recreating them.

### docker pause / unpause

```bash
docker pause mycontainer    # Freeze all processes (SIGSTOP)
docker unpause mycontainer  # Resume (SIGCONT)
```

**Why use it:** Temporarily freeze a container without stopping it.

### docker wait

```bash
# Block until container exits, prints exit code
docker wait mycontainer
```

**Why use it:** Script container workflows (e.g., run migration, wait, then run app).

### docker diff

```bash
# Show changes to filesystem since container started
docker diff mycontainer
# A /var/lib/mysql/mydb/users.ibd   (Added)
# C /var/log/app.log                (Changed)
# D /tmp/setup.tmp                  (Deleted)
```

**Why use it:** See what files the container modified during execution.

### docker update

```bash
# Update resource limits on running container
docker update --cpus=1.5 --memory=512m mycontainer
docker update --restart=always mycontainer
```

**Why use it:** Change container configuration without recreating.

### docker container prune

```bash
# Remove all stopped containers
docker container prune

# With filter
docker container prune --filter until=24h
docker container prune -f
```

**Why use it:** Batch cleanup of stopped containers.

---

## Volume Commands

### docker volume create

```bash
# Create named volume
docker volume create myvolume

# Create with labels
docker volume create --label env=production --label app=myapp mydata

# Create with driver options (NFS)
docker volume create --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.100,rw \
  --opt device=:/path/to/dir \
  nfs-volume
```

### docker volume ls

```bash
# List all volumes
docker volume ls

# Filter
docker volume ls -f dangling=true
docker volume ls -f name=app
docker volume ls -f label=env=production

# Custom format
docker volume ls --format "table {{.Name}}\t{{.Driver}}\t{{.Size}}"
```

### docker volume inspect

```bash
docker volume inspect myvolume
# Shows mountpoint, driver, labels, options
```

### docker volume rm

```bash
docker volume rm myvolume
docker volume rm -f myvolume    # Force remove even if in use
```

### docker volume prune

```bash
docker volume prune             # Remove unused volumes
docker volume prune -a          # Remove ALL unused volumes
docker volume prune -f          # Force
```

---

## Network Commands

### docker network create

```bash
# Default bridge
docker network create my-network

# Custom subnet
docker network create --subnet=172.20.0.0/16 --gateway=172.20.0.1 my-network

# With custom driver
docker network create --driver overlay my-overlay
docker network create --driver macvlan --subnet=192.168.1.0/24 -o parent=eth0 macvlan

# Internal (no external access)
docker network create --internal my-network

# Attachable overlay (standalone containers can use)
docker network create --driver overlay --attachable my-overlay

# Enable ICC (inter-container communication)
docker network create -o com.docker.network.bridge.enable_icc=false isolated

# IPAM config
docker network create --ipam-driver default --ipam-opt subnet=10.0.0.0/24 net1
```

### docker network ls

```bash
docker network ls
docker network ls -f driver=bridge
docker network ls -f name=my
```

### docker network inspect

```bash
docker network inspect my-network
# Shows subnet, gateway, connected containers, options
```

### docker network connect / disconnect

```bash
# Connect container to network
docker network connect my-network mycontainer
docker network connect --ip 172.20.0.10 my-network mycontainer
docker network connect --alias api my-network mycontainer

# Disconnect
docker network disconnect my-network mycontainer
```

### docker network prune

```bash
docker network prune
docker network prune -f
```

---

## System Commands

### docker info

```bash
docker info
# Shows: OS, architecture, CPUs, memory, storage driver, logging driver,
#        number of images/containers, server version, registry, etc.
```

### docker version

```bash
docker version
# Client version
# Server version
# API version
# Go version
# OS/Arch
```

### docker system df

Show disk usage by Docker objects.

```bash
docker system df
# TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
# Images          5         2         1.2GB     800MB (66%)
# Containers      3         2         100MB     50MB (50%)
# Local Volumes   4         2         500MB     200MB (40%)
# Build Cache     12        -         300MB     300MB (100%)

# Detailed
docker system df -v
```

### docker system prune

Remove unused Docker objects.

```bash
docker system prune                  # Containers, networks, dangling images
docker system prune -a               # + All unused images (not just dangling)
docker system prune --volumes        # + Volumes
docker system prune -af              # Force, all, no prompt

# Filter
docker system prune -af --filter until=24h
docker system prune -af --filter label!=keep
```

### docker events

Real-time events from the Docker daemon.

```bash
# Stream all events
docker events

# Filter by type
docker events --filter type=container
docker events --filter type=image
docker events --filter type=volume
docker events --filter type=network

# Filter by event
docker events --filter event=start
docker events --filter event=die
docker events --filter event=health_status:healthy

# Filter by container
docker events --filter container=myapp

# Since time
docker events --since 1h

# JSON
docker events --format "{{json .}}"
```

---

## Build Commands

### docker buildx

Multi-architecture builds.

```bash
# List builders
docker buildx ls

# Create new builder
docker buildx create --name mybuilder --use

# Build for multiple platforms
docker buildx build --platform linux/amd64,linux/arm64,linux/arm/v7 -t myuser/myapp:latest --push .

# Build with cache
docker buildx build --cache-from type=local,src=.cache --cache-to type=local,dest=.cache -t myapp .

# Bake (build multiple images from HCL)
docker buildx bake -f docker-bake.hcl
```

### docker builder

Manage build cache.

```bash
# Prune build cache
docker builder prune
docker builder prune --all
docker builder prune --filter until=24h

# Inspect build cache
docker builder inspect mybuilder

# Debug build
docker build --no-cache-filter=stage1 -t myapp .
```

---

## Compose Commands

### docker compose up

```bash
# Start services
docker compose up                      # Foreground
docker compose up -d                   # Detached
docker compose up -d --build           # Rebuild before starting
docker compose up -d --scale worker=3  # Scale a service
docker compose up -d service1 service2  # Specific services

# With profile
docker compose --profile dev up -d
```

### docker compose down

```bash
docker compose down                    # Stop and remove
docker compose down -v                 # + Remove volumes
docker compose down --rmi all          # + Remove images
docker compose down --remove-orphans   # + Remove unlisted containers
```

### docker compose build

```bash
docker compose build
docker compose build --no-cache
docker compose build --parallel
docker compose build service1
docker compose build --push
```

### docker compose logs

```bash
docker compose logs -f
docker compose logs -f service1
docker compose logs --tail=100
docker compose logs --since=5m
```

### docker compose exec

```bash
docker compose exec app bash
docker compose exec -T app npm test     # Non-TTY
docker compose exec -u root app whoami
```

### docker compose run

```bash
docker compose run --rm app npm test
docker compose run --rm -e NODE_ENV=test app npm run migrate
docker compose run --rm -p 3000:3000 app bash
```

### docker compose ps

```bash
docker compose ps
docker compose ps -a
```

### docker compose config

```bash
docker compose config
docker compose config --services
docker compose config --volumes
docker compose config --format json
docker compose config --quiet          # Validate only (exit code)
```

### docker compose top / stats

```bash
docker compose top
docker compose stats
```

---

## Registry Commands

### docker login / logout

```bash
# Docker Hub
docker login
docker login -u myuser

# Private registry
docker login registry.example.com

# With password (CI/CD — use with caution)
docker login -u myuser -p $DOCKER_PASSWORD

# Logout
docker logout
docker logout registry.example.com
```

### docker search

```bash
# Search Docker Hub
docker search nginx
docker search --limit 10 --filter stars=1000 nginx
docker search --filter is-official=true nginx
```

### docker tag

```bash
docker tag myapp:latest myuser/myapp:1.0.0
docker tag myapp:latest registry.example.com/myapp:latest
docker tag nginx:latest myregistry/nginx:custom
```

### docker push / pull

(Already covered in Image Commands)

---

## Context & Swarm Commands

### docker context

Manage multiple Docker endpoints.

```bash
# List contexts
docker context ls

# Create context
docker context create remote --docker host=tcp://192.168.1.10:2375

# Use a context
docker context use remote

# Inspect context
docker context inspect remote

# Remove context
docker context rm remote

# Export/Import
docker context export remote -- tar -c context.tar
docker context import remote context.tar
```

### docker swarm

```bash
# Initialize swarm
docker swarm init
docker swarm init --advertise-addr 192.168.1.10

# Join swarm
docker swarm join --token SWMTKN-1-... 192.168.1.10:2377

# List nodes
docker node ls

# Deploy stack
docker stack deploy -c docker-compose.yml myapp

# List services
docker service ls
docker service ps myapp

# Create service
docker service create --name web -p 80:80 nginx

# Scale service
docker service scale web=3

# Update service
docker service update --image myapp:2.0 web

# Logs for service
docker service logs web

# Leave swarm
docker swarm leave
docker swarm leave --force
```

---

## Quick Reference: Command Categories

| Category | Most Common Commands |
|----------|---------------------|
| **Image** | `build`, `pull`, `push`, `images`, `rmi`, `tag`, `history` |
| **Container** | `run`, `ps`, `stop`, `start`, `restart`, `rm`, `exec`, `logs` |
| **Volume** | `volume create`, `volume ls`, `volume rm`, `volume prune` |
| **Network** | `network create`, `network ls`, `network connect`, `network disconnect` |
| **System** | `info`, `version`, `system df`, `system prune`, `events` |
| **Build** | `buildx`, `builder prune` |
| **Compose** | `compose up`, `compose down`, `compose logs`, `compose exec` |
| **Registry** | `login`, `logout`, `tag`, `push`, `pull` |
| **Swarm** | `swarm init`, `swarm join`, `service create`, `stack deploy` |

---

## Next Steps

→ [10 — Docker Interview Questions](./10-docker-interview-questions.md)
