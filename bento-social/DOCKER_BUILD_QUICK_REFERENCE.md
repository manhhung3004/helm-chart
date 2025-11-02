# Docker Build Quick Reference

## Quick Build Commands

### Build all services at once (PowerShell)
```powershell
# Build only
.\scripts\build-docker.ps1 -Registry "your-registry.azurecr.io" -Version "latest"

# Build + Push
.\scripts\build-docker.ps1 -Registry "your-registry.azurecr.io" -Version "v1.0.0" -Push

# Manual build (one service)
docker build -t your-registry/bento-api-gateway:latest -f k8s/api-gateway/Dockerfile .
```

### Build individual services
```bash
# api-gateway
docker build -t bento-api-gateway:latest -f k8s/api-gateway/Dockerfile .

# user-service
docker build -t bento-user-service:latest -f k8s/user/Dockerfile .

# post-service
docker build -t bento-post-service:latest -f k8s/post/Dockerfile .

# notification-service
docker build -t bento-notification-service:latest -f k8s/notifications/Dockerfile .

# upload-service
docker build -t bento-upload-service:latest -f k8s/uploads/Dockerfile .

# prisma migration job
docker build -t bento-prisma-migrate:latest -f k8s/prisma-migrate-job/Dockerfile .
```

### Push to registry
```bash
# Example: Azure Container Registry
docker tag bento-api-gateway:latest myregistry.azurecr.io/bento-api-gateway:latest
docker push myregistry.azurecr.io/bento-api-gateway:latest

# Or use script
.\scripts\build-docker.ps1 -Registry "myregistry.azurecr.io" -Version "latest" -Push
```

---

## Local Testing

### Test single service
```bash
docker build -t test-api:latest -f k8s/api-gateway/Dockerfile .

docker run -p 3000:3000 \
  -e NODE_ENV=production \
  -e DATABASE_URL="postgresql://user:pass@postgres:5432/db" \
  -e JWT_SECRET="test-secret" \
  test-api:latest

# Health check
curl http://localhost:3000/healthz
```

### Test with docker-compose (all services + DB)
```bash
docker-compose -f docker-compose.test.yml up
# This spins up postgres + all services
```

---

## Image Sizes

After successful build, check image sizes:
```bash
docker images | grep bento
```

Expected:
- api-gateway: ~350MB
- user-service: ~330MB
- post-service: ~330MB
- notification-service: ~320MB
- upload-service: ~330MB
- prisma-migrate: ~280MB

---

## Dockerfile Locations

```
k8s/
├── api-gateway/Dockerfile
├── user/Dockerfile
├── post/Dockerfile
├── notifications/Dockerfile
├── uploads/Dockerfile
└── prisma-migrate-job/Dockerfile
```

---

## Build Features

✅ Multi-stage (builder + runtime)  
✅ Non-root user (uid: 1001)  
✅ Health checks included  
✅ Optimized for pnpm  
✅ Security best practices  
✅ ~70% size reduction vs single-stage

---

## Integration with Helm

Update `values.yaml` for each service:
```yaml
image:
  repository: your-registry/bento-api-gateway
  tag: "v1.0.0"
```

Deploy:
```bash
helm install api-gateway k8s/api-gateway \
  --set image.repository="your-registry/bento-api-gateway" \
  --set image.tag="v1.0.0"
```

---

## CI/CD Example (GitHub Actions)

See `.github/workflows/build-and-deploy.yml` for full pipeline setup.

Quick example:
```yaml
- name: Build Docker images
  run: |
    VERSION=${{ github.sha }}
    docker build -t my-registry/bento-api-gateway:$VERSION -f k8s/api-gateway/Dockerfile .
    docker build -t my-registry/bento-user-service:$VERSION -f k8s/user/Dockerfile .
    # ... repeat for all

- name: Push to registry
  run: docker push my-registry/bento-*:$VERSION
```

---

## Troubleshooting

**Issue:** `pnpm: command not found`  
**Solution:** Check if corepack is properly enabled in Dockerfile. It is by default.

**Issue:** Image takes too long to build  
**Solution:** Enable Docker BuildKit: `export DOCKER_BUILDKIT=1`

**Issue:** Image won't start in k8s  
**Solution:** Check health endpoint `/healthz` is implemented in NestJS app.

**Issue:** Image is too large  
**Solution:** Normal for Node.js. Check multi-stage is working (should be 300-400MB).

---

**Created:** 2025-11-02  
**Status:** Ready to build and deploy
