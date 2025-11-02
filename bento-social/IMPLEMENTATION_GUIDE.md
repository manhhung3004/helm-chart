# Helm Charts Fixes — Implementation Guide

## What I've Done

Created comprehensive audit report (`HELM_AUDIT_REPORT.md`) identifying 12 critical issues with your Helm charts. Then generated corrected templates:

### ✅ Files Created/Modified

1. **HELM_AUDIT_REPORT.md** — Full audit with 12 issues, priorities, and recommendations
2. **values-template-api-gateway.yaml** — Corrected values template (use as reference)
3. **prisma-migrate-job/templates/job.yaml** — Converted from Deployment to Job (CRITICAL FIX)
4. **api-gateway/templates/secrets.yaml** — Secret template for DATABASE_URL, JWT_SECRET
5. **api-gateway/templates/deployment.yaml** — Updated with env var injection

---

## Critical Issues Found (🔴 Must Fix)

### 1. Prisma Migration using Deployment instead of Job
**Status:** ✅ FIXED in `prisma-migrate-job/templates/job.yaml`
- Changed `kind: Deployment` → `kind: Job`
- Added `command: ["pnpm", "prisma", "migrate", "deploy"]`
- Added proper Job spec (completions, backoffLimit, ttlSecondsAfterFinished)

### 2. Image repositories hardcoded to "nginx"
**Status:** ⚠️ NEEDS YOUR ACTION
All services use `image.repository: nginx`. Need to update each values.yaml:

```yaml
# api-gateway/values.yaml
image:
  repository: your-registry.azurecr.io/bento-api-gateway
  # or: docker.io/your-org/bento-api-gateway

# user/values.yaml
image:
  repository: your-registry.azurecr.io/bento-user-service

# post/values.yaml
image:
  repository: your-registry.azurecr.io/bento-post-service

# notifications/values.yaml
image:
  repository: your-registry.azurecr.io/bento-notification-service

# uploads/values.yaml
image:
  repository: your-registry.azurecr.io/bento-upload-service

# prisma-migrate-job/values.yaml
image:
  repository: your-registry.azurecr.io/bento-api-gateway  # Use api-gateway image
```

### 3. Environment Variables not injected
**Status:** ✅ FIXED in `api-gateway/templates/deployment.yaml`
- Added `env:` section with DATABASE_URL and JWT_SECRET from Secret
- Created `api-gateway/templates/secrets.yaml`
- Need to repeat for other services

### 4. Service ports pointing to 80 instead of 3000
**Status:** ⚠️ NEEDS YOUR ACTION
Update each service's values.yaml:

```yaml
# Old (wrong)
service:
  port: 80

# New (correct)
app:
  containerPort: 3000

service:
  port: 80  # External (can stay 80)
  targetPort: 3000  # Internal
```

### 5. Health probes using "/" instead of "/healthz"
**Status:** ⚠️ NEEDS YOUR ACTION
Update each service's values.yaml:

```yaml
livenessProbe:
  httpGet:
    path: /healthz  # Instead of /
    port: 3000
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /healthz
    port: 3000
  initialDelaySeconds: 5
  periodSeconds: 5
```

**Note:** Add health endpoint to `src/app.controller.ts`:
```typescript
@Get('/healthz')
health() {
  return { status: 'ok' };
}
```

### 6. No resources requests/limits
**Status:** ⚠️ NEEDS YOUR ACTION
Add to each service's values.yaml:

```yaml
# api-gateway/values.yaml
resources:
  requests:
    cpu: "100m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"

# post/values.yaml (higher load)
resources:
  requests:
    cpu: "200m"
    memory: "512Mi"
  limits:
    cpu: "1"
    memory: "1Gi"

# notifications/values.yaml (lower load)
resources:
  requests:
    cpu: "50m"
    memory: "128Mi"
  limits:
    cpu: "250m"
    memory: "256Mi"
```

---

## Implementation Checklist

### Phase 1: Apply Critical Fixes (30 min)
- [ ] 1. Replace all `image.repository` with your registry URLs
- [ ] 2. Add `app.containerPort: 3000` to all values.yaml
- [ ] 3. Update health probe paths to `/healthz` (port: 3000)
- [ ] 4. Add resource requests/limits to each service

### Phase 2: Apply Template Fixes (20 min)
- [ ] 5. Copy corrected `api-gateway/values-template.yaml` to `values.yaml`
- [ ] 6. Apply similar env var injection to other service deployment templates
- [ ] 7. Create `secrets.yaml` template for each service
- [ ] 8. Enable autoscaling for high-load services (post, api-gateway)

### Phase 3: Add Missing Code (10 min)
- [ ] 9. Add health endpoint to `src/app.controller.ts`
- [ ] 10. Ensure all services import HealthController

### Phase 4: Validate (5 min)
- [ ] 11. Run `helm lint k8s/*` on all charts
- [ ] 12. Test with `helm template` on api-gateway chart
- [ ] 13. Dry-run with `helm install --dry-run` locally

### Phase 5: Deploy (Varies)
- [ ] 14. Update CI/CD to build and push correct image names
- [ ] 15. Deploy to minikube/kind for testing
- [ ] 16. Verify all pods are running

---

## Quick Copy-Paste Fixes

### Fix 1: Update api-gateway/values.yaml (Image + Resources)

Replace image and resources sections:
```yaml
image:
  repository: your-registry.azurecr.io/bento-api-gateway
  pullPolicy: IfNotPresent
  tag: "latest"

resources:
  requests:
    cpu: "100m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"

app:
  containerPort: 3000

livenessProbe:
  httpGet:
    path: /healthz
    port: 3000
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /healthz
    port: 3000
  initialDelaySeconds: 5
  periodSeconds: 5

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 5
  targetCPUUtilizationPercentage: 80
```

### Fix 2: Repeat for other services (user, post, notifications, uploads)

Copy the template from `values-template-api-gateway.yaml` and adapt:
- Change `repository` to your service name
- Adjust resources per service load (post > api-gateway > user > notifications)
- Keep autoscaling disabled for notifications, enabled for others

---

## Validation Commands

```bash
# Validate all charts
helm lint k8s/api-gateway
helm lint k8s/user
helm lint k8s/post
helm lint k8s/notifications
helm lint k8s/uploads
helm lint k8s/prisma-migrate-job

# Dry-run (see what will be deployed)
helm install test-api k8s/api-gateway --dry-run --debug
helm template test-api k8s/api-gateway > /tmp/api-gateway-manifest.yaml

# Check for schema validation errors
helm template test-api k8s/api-gateway --validate
```

---

## NestJS Health Endpoint

Add to `src/app.controller.ts`:

```typescript
import { Controller, Get } from '@nestjs/common';

@Controller()
export class AppController {
  // ... existing routes

  @Get('/healthz')
  healthz() {
    return { status: 'ok', timestamp: new Date() };
  }

  @Get('/readyz')
  readyz() {
    // Add DB connection check if needed
    return { ready: true };
  }
}
```

---

## File Locations & Status

```
k8s/
├── HELM_AUDIT_REPORT.md ............................ 📄 Full audit report (READ THIS!)
├── values-template-api-gateway.yaml ................ 📄 Corrected values template
├── api-gateway/
│   ├── templates/
│   │   ├── deployment.yaml ......................... ✅ UPDATED (env vars added)
│   │   ├── secrets.yaml ............................ ✅ CREATED (new)
│   │   └── service.yaml ............................ 📝 Check targetPort
│   └── values.yaml ................................ ⚠️ NEEDS UPDATE (image, resources)
├── user/
│   ├── templates/deployment.yaml ................... ⚠️ NEEDS env var injection
│   ├── templates/secrets.yaml ....................... ⚠️ NEEDS CREATION
│   └── values.yaml ................................ ⚠️ NEEDS UPDATE
├── post/
│   ├── templates/deployment.yaml ................... ⚠️ NEEDS env var injection
│   ├── templates/secrets.yaml ....................... ⚠️ NEEDS CREATION
│   └── values.yaml ................................ ⚠️ NEEDS UPDATE
├── notifications/
│   ├── templates/deployment.yaml ................... ⚠️ NEEDS env var injection
│   ├── templates/secrets.yaml ....................... ⚠️ NEEDS CREATION
│   └── values.yaml ................................ ⚠️ NEEDS UPDATE
├── uploads/
│   ├── templates/deployment.yaml ................... ⚠️ NEEDS env var injection
│   ├── templates/secrets.yaml ....................... ⚠️ NEEDS CREATION
│   └── values.yaml ................................ ⚠️ NEEDS UPDATE
└── prisma-migrate-job/
    ├── templates/
    │   ├── job.yaml ................................ ✅ CREATED (NEW - replaces deployment.yaml)
    │   ├── secrets.yaml ............................. ⚠️ NEEDS CREATION
    │   └── deployment.yaml .......................... ⚠️ DELETE THIS (replaced by job.yaml)
    └── values.yaml ................................ ⚠️ NEEDS UPDATE (image, db-url secret)
```

---

## Next Steps

### Immediate (Before Deploy)
1. Read `HELM_AUDIT_REPORT.md` in detail
2. Update all `values.yaml` files with:
   - Correct image repositories
   - Resource requests/limits
   - Health probe paths and ports
   - Environment variables
3. Add `secrets.yaml` templates to each service
4. Add health endpoints to NestJS app
5. Run `helm lint` on all charts

### Then
6. Test locally with `helm install --dry-run`
7. Deploy to minikube/kind
8. Verify all pods are Running
9. Check service discovery works (curl between services)
10. Test Prisma migration job runs successfully

### Finally
11. Set up CI/CD to build correct images
12. Deploy to production

---

**Generated:** 2025-11-02  
**Status:** Implementation guide ready — follow checklist above to complete fixes
