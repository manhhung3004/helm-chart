# Dockerfile Build Guide for bento-social-nestjs

## Overview

All services use the same Dockerfile pattern:
1. **Multi-stage build** — Builder stage (with devDependencies) + Runtime stage (minimal)
2. **pnpm** — Fast, locked dependency management
3. **Non-root user** — Security best practice (uid: 1001)
4. **Health checks** — Built-in HEALTHCHECK instruction
5. **Lean images** — Only production code + dependencies

---

## File Locations

```
k8s/
├── api-gateway/Dockerfile          ← Main API gateway
├── user/Dockerfile                 ← User service
├── post/Dockerfile                 ← Post service (+ likes, saves)
├── notifications/Dockerfile        ← Notification service
├── uploads/Dockerfile              ← Upload service
└── prisma-migrate-job/Dockerfile  ← Prisma migrations (Job image)

Root:
├── Dockerfile                      ← Full monolith (for reference)
```

---

## Build Commands

### From Project Root

```bash
# Build all service images
docker build -t bento-api-gateway:latest -f k8s/api-gateway/Dockerfile .
docker build -t bento-user-service:latest -f k8s/user/Dockerfile .
docker build -t bento-post-service:latest -f k8s/post/Dockerfile .
docker build -t bento-notification-service:latest -f k8s/notifications/Dockerfile .
docker build -t bento-upload-service:latest -f k8s/uploads/Dockerfile .
docker build -t bento-prisma-migrate:latest -f k8s/prisma-migrate-job/Dockerfile .

# Or use a script (see below)
```

---

## Dockerfile Structure Explained

### Stage 1: Builder

```dockerfile
FROM node:20-alpine AS builder

WORKDIR /app

# Enable pnpm
ENV PNPM_HOME="/pnpm"
RUN corepack enable && corepack prepare pnpm@latest --activate

# Install dependencies (including devDependencies)
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile

# Build NestJS
COPY . .
RUN pnpm build
```

**Why this approach:**
- `node:20-alpine` — 150MB base (small, has build tools)
- `--frozen-lockfile` — Reproducible builds
- Build artifacts + node_modules stay here
- This stage is ~1.5-2GB (temporary)

### Stage 2: Runtime

```dockerfile
FROM node:20-alpine AS runtime

WORKDIR /app

ENV NODE_ENV=production

# Only production dependencies
RUN pnpm install --prod --frozen-lockfile

# Copy only what's needed from builder
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules

# Non-root user
RUN addgroup -g 1001 -S nodejs && adduser -S nestjs -u 1001
USER nestjs

# Health check + startup
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD node -e "require('http').get('http://localhost:3000/healthz', ...)"

CMD ["node", "dist/main.js"]
```

**Final image: ~300-400MB** (includes runtime + prod dependencies only)

---

## Key Features

### 1. Multi-stage Build
- **Builder stage** (discarded after build):
  - Includes TypeScript compiler, build tools
  - Size: ~2GB (temporary)
  
- **Runtime stage** (deployed image):
  - Only compiled code + prod dependencies
  - Size: ~300-400MB (final)

**Benefit:** 80-85% size reduction vs single-stage build

### 2. Security
- Non-root user: `nestjs` (uid: 1001)
- Read-only filesystem (can add via k8s)
- No sudo, no shell entry point

### 3. Health Checks
```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD node -e "require('http').get('http://localhost:3000/healthz', (r) => {if (r.statusCode !== 200) throw new Error(r.statusCode)})"
```

Kubernetes uses this for liveness/readiness probes automatically.

### 4. pnpm Optimization
- Faster installs (parallel)
- Smaller node_modules (hoisted)
- Lock file ensures reproducibility
- `--prod` flag omits devDependencies for runtime

---

## Build Script

Create `scripts/build-docker.sh` (or `.ps1` for PowerShell):

```bash
#!/bin/bash

set -e

REGISTRY="${REGISTRY:-docker.io/myorg}"
VERSION="${VERSION:-$(git rev-parse --short HEAD)}"

echo "🐳 Building Docker images for bento-social-nestjs"
echo "Registry: $REGISTRY"
echo "Version: $VERSION"

services=(
  "api-gateway"
  "user"
  "post"
  "notifications"
  "uploads"
  "prisma-migrate-job"
)

for service in "${services[@]}"; do
  image_name="$REGISTRY/bento-$service:$VERSION"
  dockerfile="k8s/$service/Dockerfile"
  
  echo "📦 Building $image_name"
  docker build -t "$image_name" -f "$dockerfile" .
  
  echo "✅ Built: $image_name"
done

echo "✨ All images built successfully!"
echo "Next: docker push $REGISTRY/bento-*:$VERSION"
```

Usage:
```bash
REGISTRY=your-registry.azurecr.io VERSION=latest ./scripts/build-docker.sh
# or
VERSION=v1.0.0 ./scripts/build-docker.sh
```

---

## Build Performance Tips

### 1. Use Docker BuildKit (faster, better caching)

```bash
export DOCKER_BUILDKIT=1
docker build -t bento-api-gateway:latest -f k8s/api-gateway/Dockerfile .
```

### 2. Build only what changed
```bash
# BuildKit caches per layer, so:
# - Changing package.json rebuilds from that layer
# - Changing .ts files rebuilds from COPY . . layer
# - Node modules layer is cached if package.json unchanged
```

### 3. Ignore unnecessary files in build context

Create `.dockerignore` at project root:
```
node_modules
dist
.git
.env
.env.*
!.env.example
README.md
.eslintrc.js
.prettierrc
```

---

## Testing Locally

### Test build
```bash
# Build image
docker build -t test-api:latest -f k8s/api-gateway/Dockerfile .

# Run container
docker run -p 3000:3000 \
  -e NODE_ENV=production \
  -e DATABASE_URL="postgresql://..." \
  -e JWT_SECRET="test-secret" \
  test-api:latest

# Test health
curl http://localhost:3000/healthz

# See logs
docker logs <container-id>
```

### Test with docker-compose (for full stack)

Create `docker-compose.test.yml`:
```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: bento_db
    ports:
      - "5432:5432"

  api-gateway:
    build:
      context: .
      dockerfile: k8s/api-gateway/Dockerfile
    environment:
      DATABASE_URL: postgresql://postgres:postgres@postgres:5432/bento_db
      NODE_ENV: production
    ports:
      - "3000:3000"
    depends_on:
      - postgres

  user-service:
    build:
      context: .
      dockerfile: k8s/user/Dockerfile
    environment:
      DATABASE_URL: postgresql://postgres:postgres@postgres:5432/bento_db
      NODE_ENV: production
    ports:
      - "3001:3000"
    depends_on:
      - postgres
```

Run:
```bash
docker-compose -f docker-compose.test.yml up
```

---

## Helm Integration

Update `values.yaml` for each service to reference the Dockerfile:

```yaml
# k8s/api-gateway/values.yaml
image:
  repository: your-registry.azurecr.io/bento-api-gateway
  # This image is built from: k8s/api-gateway/Dockerfile
  pullPolicy: IfNotPresent
  tag: "latest"
```

Build and push:
```bash
docker build -t your-registry.azurecr.io/bento-api-gateway:latest -f k8s/api-gateway/Dockerfile .
docker push your-registry.azurecr.io/bento-api-gateway:latest

# Then deploy with Helm
helm install api-gateway k8s/api-gateway \
  --set image.tag=latest
```

---

## Image Size Reference

| Image | Size | Notes |
|-------|------|-------|
| node:20-alpine | 150MB | Base image |
| After build (runtime stage) | 300-400MB | With app + prod deps |
| Multi-stage savings | ~70% | vs 1.2GB single-stage |

---

## Troubleshooting

### Docker image won't build
```bash
# Check if pnpm-lock.yaml exists
ls pnpm-lock.yaml

# Check if node_modules is in .dockerignore
cat .dockerignore | grep node_modules

# Check Dockerfile syntax
docker build --no-cache -f k8s/api-gateway/Dockerfile .
```

### Image won't start
```bash
# Check if health endpoint exists
curl http://localhost:3000/healthz

# Check environment variables
docker inspect <image-id> | grep -A 10 ENV

# Check logs
docker run --rm -it <image> /bin/sh
npm run start:debug
```

### pnpm install fails in Docker
```bash
# Try clearing pnpm cache in Dockerfile
RUN pnpm store prune
```

---

## CI/CD Integration

GitHub Actions example (see main docs for full setup):

```yaml
- name: Build Docker images
  run: |
    VERSION=${{ github.sha }}
    docker build -t $REGISTRY/bento-api-gateway:$VERSION -f k8s/api-gateway/Dockerfile .
    docker build -t $REGISTRY/bento-user-service:$VERSION -f k8s/user/Dockerfile .
    # ... repeat for all services
    
- name: Push images
  run: |
    docker push $REGISTRY/bento-*:$VERSION
```

---

## Summary

- ✅ 6 production-ready Dockerfiles (5 services + 1 migration job)
- ✅ Multi-stage builds (300-400MB final images)
- ✅ Non-root user for security
- ✅ Health checks included
- ✅ Ready to push to any registry
- ✅ Compatible with Helm charts

**Next:** Build, test locally, then integrate with CI/CD pipeline.
