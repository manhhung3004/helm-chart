# Helm Charts Audit Report

## Summary
✅ **Good** — You have proper Helm chart scaffolding with all 6 service charts (api-gateway, user, post, notifications, uploads, prisma-migrate-job). The structure is sound.

❌ **Critical Issues** — Most `values.yaml` files are not customized for your NestJS microservices; they contain boilerplate placeholder values (nginx image, port 80, etc.). The `prisma-migrate-job` should use a **Job** kind, not Deployment.

---

## Issues Found

### 1. **Prisma Migration Job is using Deployment instead of Job** ⚠️ CRITICAL

**File:** `k8s/prisma-migrate-job/templates/deployment.yaml`

**Problem:**
- The file is named `deployment.yaml` but migration should use `batch/v1/Job`, not `apps/v1/Deployment`.
- A Deployment will keep restarting the migration container, causing repeated migration attempts.
- Should be a one-time Job that completes and stops.

**Fix:**
- Rename template to `job.yaml`
- Change `kind: Deployment` → `kind: Job`
- Adjust spec structure to match Job template (add `backoffLimit`, `completions`, etc.)

---

### 2. **Image Repository is hardcoded to "nginx" everywhere** ⚠️ HIGH PRIORITY

**Files affected:**
- `k8s/api-gateway/values.yaml`
- `k8s/user/values.yaml`
- `k8s/post/values.yaml`
- `k8s/notifications/values.yaml`
- `k8s/uploads/values.yaml`
- `k8s/prisma-migrate-job/values.yaml`

**Problem:**
```yaml
image:
  repository: nginx  # ❌ Wrong! Should be your container registry
```

**Fix:**
Replace with your actual registry and service names:
```yaml
# api-gateway/values.yaml
image:
  repository: your-registry.azurecr.io/bento-api-gateway  # or docker.io/your-org/bento-api-gateway
  tag: "latest"  # Should be overridden at deploy time with git commit SHA

# user/values.yaml
image:
  repository: your-registry.azurecr.io/bento-user-service
  tag: "latest"

# post/values.yaml
image:
  repository: your-registry.azurecr.io/bento-post-service
  tag: "latest"

# notifications/values.yaml
image:
  repository: your-registry.azurecr.io/bento-notification-service
  tag: "latest"

# uploads/values.yaml
image:
  repository: your-registry.azurecr.io/bento-upload-service
  tag: "latest"

# prisma-migrate-job/values.yaml
image:
  repository: your-registry.azurecr.io/bento-api-gateway  # Use api-gateway image since prisma is there
  tag: "latest"
```

---

### 3. **Service Port is hardcoded to 80** ⚠️ MEDIUM

**Problem:**
All services define `service.port: 80`, but NestJS apps typically run on port 3000 internally.

```yaml
# current
service:
  port: 80  # ❌ Wrong

# deployment.yaml also exposes
containerPort: {{ .Values.service.port }}  # This becomes 80 → 3000 mismatch
```

**Fix:**
Update `values.yaml` for each service:
```yaml
service:
  port: 3000  # or keep as 80 and set containerPort separately

# Better practice:
containerPort: 3000  # Internal
service:
  port: 80  # External
```

**Or separate them in values:**
```yaml
app:
  containerPort: 3000

service:
  port: 80
  targetPort: 3000  # Added in service.yaml template
```

---

### 4. **Health Probes use "/" instead of "/healthz"** ⚠️ MEDIUM

**Files affected:** All `values.yaml`

**Problem:**
```yaml
livenessProbe:
  httpGet:
    path: /        # ❌ Likely not correct for NestJS
    port: http
readinessProbe:
  httpGet:
    path: /        # ❌ Same
    port: http
```

**Fix:**
```yaml
livenessProbe:
  httpGet:
    path: /healthz  # or /api/health
    port: 3000
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5

readinessProbe:
  httpGet:
    path: /healthz
    port: 3000
  initialDelaySeconds: 5
  periodSeconds: 5
  timeoutSeconds: 3
```

**Note:** Ensure your NestJS app has a health endpoint. See below for suggested code.

---

### 5. **Missing Environment Variables in Deployment Templates** ⚠️ HIGH

**Files affected:** All `templates/deployment.yaml`

**Problem:**
Deployments don't inject secrets or config (DATABASE_URL, API_KEYS, etc.):
```yaml
containers:
  - name: {{ .Chart.Name }}
    # ❌ No env: section for DATABASE_URL, JWT_SECRET, etc.
```

**Fix:**
Add `env:` section to `templates/deployment.yaml`:
```yaml
env:
  - name: NODE_ENV
    value: "production"
  - name: DATABASE_URL
    valueFrom:
      secretKeyRef:
        name: {{ include "api-gateway.fullname" . }}-secrets
        key: database-url
  - name: JWT_SECRET
    valueFrom:
      secretKeyRef:
        name: {{ include "api-gateway.fullname" . }}-secrets
        key: jwt-secret
  # Add other required env vars based on your .env
```

Also create `k8s/<service>/templates/secret.yaml`:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "api-gateway.fullname" . }}-secrets
type: Opaque
stringData:
  database-url: {{ .Values.secrets.databaseUrl | quote }}
  jwt-secret: {{ .Values.secrets.jwtSecret | quote }}
```

And add to `values.yaml`:
```yaml
secrets:
  databaseUrl: "postgresql://user:pass@postgres:5432/db"
  jwtSecret: "your-secret-key"
```

---

### 6. **Missing Resource Requests/Limits** ⚠️ MEDIUM

**Problem:**
```yaml
resources: {}  # ❌ No requests/limits defined
```

**Fix:**
Per service in `values.yaml`:
```yaml
# api-gateway/values.yaml
resources:
  requests:
    cpu: "100m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"

# post/values.yaml (higher load expected)
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

### 7. **Autoscaling is disabled** ⚠️ MEDIUM

**Problem:**
```yaml
autoscaling:
  enabled: false  # ❌ Consider enabling
```

**Fix:**
Enable for services that need it:
```yaml
# post/values.yaml (high traffic expected)
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

# api-gateway/values.yaml
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 5
  targetCPUUtilizationPercentage: 80

# notifications/values.yaml (event-driven, lighter)
autoscaling:
  enabled: false
  # Or keep simple with 1 replica
```

---

### 8. **Missing Service account and RBAC** ⚠️ LOW-MEDIUM

**Problem:**
ServiceAccount is created but no RBAC (Role, RoleBinding) for prisma-migrate-job:
```yaml
serviceAccount:
  create: true  # Created but no permissions attached
```

**Fix for prisma-migrate-job:**
Add `k8s/prisma-migrate-job/templates/rbac.yaml`:
```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: {{ include "prisma-migrate-job.fullname" . }}
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: {{ include "prisma-migrate-job.fullname" . }}
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: {{ include "prisma-migrate-job.fullname" . }}
subjects:
  - kind: ServiceAccount
    name: {{ include "prisma-migrate-job.serviceAccountName" . }}
```

---

### 9. **No Secrets/ConfigMap templates** ⚠️ HIGH

**Problem:**
Charts don't define how secrets are created.

**Fix:**
- Create `k8s/<service>/templates/secrets.yaml` (if storing sensitive config)
- Create `k8s/<service>/templates/configmap.yaml` (for non-sensitive config)

Example:
```yaml
# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "api-gateway.fullname" . }}-config
data:
  LOG_LEVEL: {{ .Values.config.logLevel | quote }}
  API_PREFIX: {{ .Values.config.apiPrefix | quote }}
```

---

### 10. **Ingress is disabled and unconfigured** ⚠️ MEDIUM

**Problem:**
```yaml
ingress:
  enabled: false
  # No real ingress rules
```

**Fix:**
For api-gateway, enable ingress:
```yaml
# api-gateway/values.yaml
ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
    - host: api.yourdomain.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: api-tls
      hosts:
        - api.yourdomain.com
```

---

### 11. **HTTPRoute templates are empty** ⚠️ LOW

**Problem:**
`httproute.yaml` templates exist but contain generic placeholders.

**Fix:**
Either:
- A) Configure HTTPRoute if using Gateway API (Istio, Traefik v2.1+):
  ```yaml
  httpRoute:
    enabled: true
    parentRefs:
      - name: main-gateway
        namespace: ingress-system
    hostnames:
      - api.yourdomain.com
    rules:
      - matches:
          - path:
              type: PathPrefix
              value: /api
  ```
- B) Remove HTTPRoute if using standard Ingress.

---

### 12. **Prisma Migration Job is missing command & args** ⚠️ CRITICAL

**Problem:**
Job doesn't specify what command to run for migration:
```yaml
# deployment.yaml is used as Job, but no migration command defined
```

**Fix:**
In `k8s/prisma-migrate-job/templates/job.yaml`:
```yaml
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          command: ["pnpm", "prisma", "migrate", "deploy"]  # ← Add this
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: {{ include "prisma-migrate-job.fullname" . }}-secrets
                  key: database-url
      restartPolicy: Never
  backoffLimit: 3
```

---

## Recommendations (Priority Order)

### 🔴 Must Fix Before Deploy
1. **Change prisma-migrate-job from Deployment to Job** (file type + spec)
2. **Update all image repositories** (nginx → your actual images)
3. **Add environment variables** to all deployment templates
4. **Create Secret templates** for each service (DATABASE_URL, JWT_SECRET, etc.)
5. **Fix port mappings** (containerPort 3000, service port 80 or 3000)
6. **Add health endpoint** to NestJS app if missing

### 🟡 Should Fix Soon
7. Add resource requests/limits
8. Enable autoscaling for high-load services
9. Configure ingress for api-gateway
10. Fix health probe paths to real endpoints

### 🟢 Nice to Have
11. Add RBAC for prisma job
12. Create ConfigMaps for app config
13. Add HTTPRoute if using Gateway API

---

## Required NestJS Code Addition

Your NestJS app needs a health endpoint. Add to `src/app.controller.ts` or create `src/health.controller.ts`:

```typescript
import { Controller, Get } from '@nestjs/common';

@Controller()
export class HealthController {
  @Get('/healthz')
  health() {
    return { status: 'ok' };
  }

  @Get('/readyz')
  readiness() {
    // Can add DB connection check here
    return { status: 'ready' };
  }
}
```

Add to `app.module.ts`:
```typescript
import { HealthController } from './health.controller';

@Module({
  controllers: [AppController, HealthController],
  // ...
})
export class AppModule {}
```

---

## Next Steps

1. **Create improved values.yaml templates** for each service (I can provide)
2. **Convert prisma-migrate-job to use Job kind** (I can provide)
3. **Create Secret template files** (I can provide)
4. **Test locally with minikube/kind** before production
5. **Set up CI/CD to override image tags** at deployment time

---

## Quick Commands to Validate Charts

```bash
# Validate chart syntax
helm lint k8s/api-gateway
helm lint k8s/user
helm lint k8s/post
helm lint k8s/notifications
helm lint k8s/uploads
helm lint k8s/prisma-migrate-job

# Dry-run to see what would be deployed
helm install my-api-gateway k8s/api-gateway --dry-run --debug

# Validate templates
helm template my-api-gateway k8s/api-gateway
```

---

**Created:** 2025-11-02  
**Status:** Review and implement fixes before production deployment
