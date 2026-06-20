# 08 — Dockerfile Recipes & Patterns

> Production-ready Dockerfiles for every language and pattern

---

## Table of Contents

1. [Node.js Dockerfiles](#nodejs-dockerfiles)
2. [Python Dockerfiles](#python-dockerfiles)
3. [Go Dockerfiles](#go-dockerfiles)
4. [Rust Dockerfiles](#rust-dockerfiles)
5. [Java / Spring Boot Dockerfiles](#java--spring-boot-dockerfiles)
6. [Development Dockerfiles](#development-dockerfiles)
7. [Dockerfile Anti-Patterns](#dockerfile-anti-patterns)
8. [BuildKit Features](#buildkit-features)

---

## Node.js Dockerfiles

### Production — Full-Featured

```dockerfile
# === Stage 1: Build ===
FROM node:20-alpine AS builder
WORKDIR /app

# Dependencies first (layer caching)
COPY package.json package-lock.json ./
RUN npm ci --omit=dev

# Build the app
COPY tsconfig.json ./
COPY src/ ./src/
RUN npm run build

# === Stage 2: Production Dependencies ===
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev --ignore-scripts

# === Stage 3: Runtime ===
FROM node:20-alpine
WORKDIR /app

# Security: non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Copy production dependencies
COPY --from=deps --chown=appuser:appgroup /app/node_modules ./node_modules

# Copy built application
COPY --from=builder --chown=appuser:appgroup /app/dist ./dist

# Copy package.json for script references
COPY --chown=appuser:appgroup package.json ./

# Security: read-only filesystem
USER appuser
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/health', r => r.statusCode === 200 ? process.exit(0) : process.exit(1))"

ENV NODE_ENV=production
CMD ["node", "dist/main.js"]
```

### Production — Minimal (PnPM + Node)

```dockerfile
FROM node:20-alpine AS base
ENV PNPM_HOME=/pnpm
ENV PATH=$PNPM_HOME:$PATH
RUN corepack enable

FROM base AS deps
WORKDIR /app
COPY pnpm-lock.yaml ./
RUN pnpm fetch --prod

FROM base AS build
WORKDIR /app
COPY pnpm-lock.yaml package.json ./
RUN pnpm fetch
COPY . .
RUN pnpm run build

FROM base
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
USER node
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

### Express + TypeScript

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json tsconfig.json ./
RUN npm ci
COPY src/ ./src/
RUN npm run build

FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev && npm cache clean --force
COPY --from=builder /app/dist ./dist
USER node
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1
CMD ["node", "dist/index.js"]
```

### Next.js

```dockerfile
FROM node:20-alpine AS base
WORKDIR /app

FROM base AS deps
COPY package.json package-lock.json* ./
RUN npm ci

FROM base AS builder
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ENV NEXT_TELEMETRY_DISABLED=1
RUN npm run build

FROM base AS runner
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nextjs
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static
USER nextjs
EXPOSE 3000
ENV PORT=3000
ENV NODE_ENV=production
CMD ["node", "server.js"]
```

---

## Python Dockerfiles

### FastAPI / Django (Production)

```dockerfile
# === Stage 1: Builder ===
FROM python:3.12-slim AS builder
WORKDIR /app

# Install build dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc build-essential && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt .
RUN pip install --user --no-cache-dirs -r requirements.txt

# === Stage 2: Runtime ===
FROM python:3.12-slim
WORKDIR /app

# Security: non-root user
RUN addgroup --system app && adduser --system --ingroup app app

# Copy installed packages from builder
COPY --from=builder /root/.local /root/.local

# Copy application
COPY --chown=app:app . .

# Make sure scripts in .local are usable
ENV PATH=/root/.local/bin:$PATH

USER app
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Flask with Gunicorn

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dirs -r requirements.txt

FROM python:3.12-slim
WORKDIR /app
RUN addgroup --system app && adduser --system --ingroup app app
COPY --from=builder /root/.local /root/.local
COPY --chown=app:app . .
ENV PATH=/root/.local/bin:$PATH
USER app
EXPOSE 8000
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "4", "app:app"]
```

---

## Go Dockerfiles

### Go — Tiny Binary (scratch)

```dockerfile
# === Stage 1: Build ===
FROM golang:1.22-alpine AS builder
WORKDIR /app

# Cache dependencies
COPY go.mod go.sum ./
RUN go mod download

# Build with optimizations
COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags="-s -w" -o /app/server ./cmd/server

# === Stage 2: Runtime (scratch) ===
FROM scratch

# Security: no shell, no package manager, nothing but the binary
COPY --from=builder /app/server /server

# For HTTPS/TLS
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Timezone data (optional)
COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo

EXPOSE 8080
ENTRYPOINT ["/server"]
```

### Go — With Debugging (Alpine)

```dockerfile
# === Stage 1: Build ===
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o /app/server .

# === Stage 2: Runtime (Alpine — has shell for debugging) ===
FROM alpine:3.19
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
COPY --from=builder /app/server /server
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
USER appuser
EXPOSE 8080
ENTRYPOINT ["/server"]
```

---

## Rust Dockerfiles

### Rust — Minimal (scratch)

```dockerfile
# === Stage 1: Build ===
FROM rust:1.77-alpine AS builder
WORKDIR /app

# Install musl target for static linking
RUN apk add --no-cache musl-dev
RUN rustup target add x86_64-unknown-linux-musl

# Cache dependencies (separate from source code)
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo "fn main() {}" > src/main.rs
RUN cargo build --target x86_64-unknown-linux-musl --release 2>/dev/null || true

# Build the real app
COPY src/ ./src/
RUN cargo build --target x86_64-unknown-linux-musl --release

# === Stage 2: Runtime ===
FROM scratch
COPY --from=builder /app/target/x86_64-unknown-linux-musl/release/myapp /myapp
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
EXPOSE 8080
ENTRYPOINT ["/myapp"]
```

### Rust — With Debugging

```dockerfile
FROM rust:1.77-slim AS builder
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo "fn main() {}" > src/main.rs
RUN cargo build --release 2>/dev/null || true
COPY src/ ./src/
RUN cargo build --release

FROM debian:bookworm-slim
RUN addgroup --system app && adduser --system --ingroup app app
COPY --from=builder /app/target/release/myapp /usr/local/bin/myapp
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
USER app
EXPOSE 8080
ENTRYPOINT ["myapp"]
```

---

## Java / Spring Boot Dockerfiles

### Spring Boot with Maven

```dockerfile
# === Stage 1: Build ===
FROM maven:3.9-eclipse-temurin-21-alpine AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline -B
COPY src/ ./src/
RUN mvn package -DskipTests -B

# === Stage 2: Runtime ===
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
COPY --from=builder --chown=appuser:appgroup /app/target/*.jar app.jar
USER appuser
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/actuator/health || exit 1
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Spring Boot with Gradle

```dockerfile
FROM gradle:8-jdk21-alpine AS builder
WORKDIR /app
COPY build.gradle* settings.gradle* ./
RUN gradle dependencies --no-daemon 2>/dev/null || true
COPY src/ ./src/
RUN gradle bootJar --no-daemon

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
COPY --from=builder --chown=appuser:appgroup /app/build/libs/*.jar app.jar
USER appuser
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Using Spring Boot's Layered JAR (Optimized)

```dockerfile
FROM eclipse-temurin:21-jre-alpine AS builder
WORKDIR /app
COPY target/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Copy layers (dependencies change less often than application)
COPY --from=builder --chown=appuser:appgroup /app/dependencies/ ./
COPY --from=builder --chown=appuser:appgroup /app/spring-boot-loader/ ./
COPY --from=builder --chown=appuser:appgroup /app/snapshot-dependencies/ ./
COPY --from=builder --chown=appuser:appgroup /app/application/ ./

USER appuser
EXPOSE 8080
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

---

## Development Dockerfiles

### Node.js — Hot Reload

```dockerfile
FROM node:20-alpine
WORKDIR /app

# Copy package files first
COPY package*.json ./
RUN npm ci

# Copy source code
COPY . .

# Security: non-root
USER node

EXPOSE 3000

# Development command with hot-reload
CMD ["npm", "run", "dev"]
```

```yaml
# docker-compose.dev.yml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.dev
    ports:
      - "3000:3000"
    volumes:
      - ./src:/app/src          # Hot-reload
      - ./package.json:/app/package.json
    environment:
      - NODE_ENV=development
      - CHOKIDAR_USEPOLLING=true  # For file watching in Docker
```

### Python — Hot Reload

```dockerfile
FROM python:3.12-slim
WORKDIR /app
RUN addgroup --system app && adduser --system --ingroup app app

# Install dependencies first
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .
USER app
EXPOSE 8000

# Hot-reload with --reload flag
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]
```

### Go — Hot Reload (Air)

```dockerfile
FROM golang:1.22-alpine AS dev
WORKDIR /app

# Install Air for hot-reload
RUN go install github.com/air-verse/air@latest

# Cache dependencies
COPY go.mod go.sum ./
RUN go mod download

# Use Air config
COPY .air.toml ./
CMD ["air"]
```

---

## Dockerfile Anti-Patterns

### 1. Using :latest Tag

```dockerfile
# ❌ BAD
FROM node:latest

# ✅ GOOD
FROM node:20.11-alpine
```

**Why:** `:latest` changes without warning. Builds are not reproducible.

### 2. Installing Unnecessary Packages

```dockerfile
# ❌ BAD
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y curl wget git vim nano build-essential

# ✅ GOOD
FROM alpine:3.19
RUN apk add --no-cache curl
```

**Why:** Every package is a security risk and increases image size.

### 3. Copying Everything

```dockerfile
# ❌ BAD
COPY . .

# ✅ GOOD
COPY package.json package-lock.json ./
RUN npm ci
COPY src/ ./src/
COPY public/ ./public/
```

**Why:** Docker sends the entire build context to the daemon. Without `.dockerignore`, you might include `node_modules`, `.git`, `.env` — slowing builds and potentially leaking secrets.

### 4. Multiple RUN Statements

```dockerfile
# ❌ BAD (4 layers, all unnecessary)
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y git
RUN rm -rf /var/lib/apt/lists/*

# ✅ GOOD (1 layer, one cached instruction)
RUN apt-get update && \
    apt-get install -y curl git && \
    rm -rf /var/lib/apt/lists/*
```

**Why:** Each `RUN` creates a layer. More layers = larger image. Cleanup in the same layer keeps it small.

### 5. Wrong Layer Order

```dockerfile
# ❌ BAD (source code copied before deps)
COPY . .
RUN npm ci
# Any file change → npm ci runs again

# ✅ GOOD (deps copied before source)
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
# npm ci only runs when package.json changes
```

**Why:** Order layers by change frequency. Dependencies change less often than source code.

### 6. Running as Root

```dockerfile
# ❌ BAD
FROM node:20-alpine
WORKDIR /app
COPY . .
CMD ["node", "app.js"]
# Result: runs as root

# ✅ GOOD
FROM node:20-alpine
WORKDIR /app
COPY . .
RUN chown -R node:node /app
USER node
CMD ["node", "app.js"]
```

**Why:** If an attacker compromises the app, they have root access.

### 7. Not Using Multi-Stage Builds

```dockerfile
# ❌ BAD — 1.2GB image with build tools
FROM node:20
WORKDIR /app
COPY . .
RUN npm ci && npm run build
EXPOSE 3000
CMD ["node", "dist/server.js"]

# ✅ GOOD — 150MB without build tools
FROM node:20-alpine AS builder
WORKDIR /app
COPY . .
RUN npm ci && npm run build

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

### 8. Hardcoding Environment Variables

```dockerfile
# ❌ BAD
ENV DB_PASSWORD=supersecret
ENV API_KEY=abc123

# ✅ GOOD (values set at runtime)
ENV DB_PASSWORD=${DB_PASSWORD}
ENV API_KEY=${API_KEY}
# docker run -e DB_PASSWORD=realpassword -e API_KEY=realkey app
```

**Why:** `docker history` reveals ALL environment variables baked into the image.

### 9. Using ADD for URLs

```dockerfile
# ❌ BAD (ADD for URLs)
ADD https://example.com/file.tar.gz /tmp/

# ✅ GOOD (use curl in RUN)
RUN curl -fsSL https://example.com/file.tar.gz -o /tmp/file.tar.gz && \
    tar -xzf /tmp/file.tar.gz -C /opt && \
    rm /tmp/file.tar.gz
```

**Why:** `ADD` for URLs doesn't cache well and doesn't automatically clean up.

### 10. Not Pinning OS Package Versions

```dockerfile
# ❌ BAD
RUN apt-get install -y curl

# ✅ GOOD
RUN apt-get install -y curl=7.88.1-10+deb12u1
```

**Why:** Unpinned versions can change between builds, causing unexpected behavior. Use `apt list --installed` to find versions.

---

## BuildKit Features

BuildKit is Docker's next-generation build system (enabled by default in Docker 24+).

### Enable BuildKit

```bash
export DOCKER_BUILDKIT=1
export COMPOSE_DOCKER_CLI_BUILD=1

# Or set in /etc/docker/daemon.json:
# { "features": { "buildkit": true } }
```

### Cache Mounts

```dockerfile
# Cache npm packages across builds
RUN --mount=type=cache,target=/root/.npm \
    npm ci

# Cache apt packages
RUN --mount=type=cache,target=/var/cache/apt \
    apt-get update && apt-get install -y python3

# Cache Go modules
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download

# Cache pip packages
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt
```

### Secret Mounts (Build-Time Secrets)

```dockerfile
# Dockerfile
RUN --mount=type=secret,id=npmrc \
    cp /run/secrets/npmrc ~/.npmrc && npm ci

RUN --mount=type=secret,id=ssh \
    --mount=type=ssh \
    npm install git+ssh://git@github.com/private/repo.git
```

```bash
docker build \
  --secret id=npmrc,src=./.npmrc \
  --ssh default \
  -t myapp .
```

### SSH Agent Forwarding

```dockerfile
# Dockerfile
RUN --mount=type=ssh \
    git clone git@github.com:org/private-repo.git /app/vendor
```

```bash
docker build --ssh default -t myapp .
```

### Inline Cache

```yaml
# docker buildx build --cache-to/--cache-from
docker buildx build \
  --cache-to type=registry,ref=myapp:cache,mode=max \
  --cache-from type=registry,ref=myapp:cache \
  -t myapp:latest .
```

### Heredoc Support

```dockerfile
# Multi-line RUN scripts without escaping
RUN <<EOF
apt-get update
apt-get install -y curl git
apt-get clean
rm -rf /var/lib/apt/lists/*
EOF
```

### Output Files

```dockerfile
# Export a file from the build (not just an image)
FROM alpine AS certs
RUN apk add --no-cache ca-certificates
# Only the certs directory is exported
```

```bash
docker build --output type=local,dest=./output .
```

---

## Summary

| Pattern | Best For | Size |
|---------|----------|------|
| **Node.js** | Multi-stage with deps stage | ~150MB |
| **Python** | Multi-stage, pip --user | ~130MB |
| **Go (scratch)** | Static binary, no deps | ~12MB |
| **Rust (scratch)** | Static binary, musl target | ~8MB |
| **Spring Boot** | Layered JAR for cache | ~250MB |
| **Dev** | Hot-reload with bind mounts | ~150MB |

---

## Next Steps

→ [09 — Docker Commands Reference](./09-docker-commands-reference.md)
