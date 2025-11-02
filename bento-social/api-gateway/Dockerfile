# Multi-stage Dockerfile for api-gateway service
# Build: node:20-alpine with build dependencies
# Runtime: minimal alpine image

# ============================================
# Stage 1: Builder
# ============================================
FROM node:20-alpine AS builder

WORKDIR /app

# Enable pnpm via corepack
ENV PNPM_HOME="/pnpm"
RUN corepack enable && corepack prepare pnpm@latest --activate

# Copy package files
COPY package.json pnpm-lock.yaml ./

# Install dependencies (frozen lockfile for reproducibility)
RUN pnpm install --frozen-lockfile

# Copy source code
COPY . .

# Build NestJS application
RUN pnpm build

# ============================================
# Stage 2: Runtime
# ============================================
FROM node:20-alpine AS runtime

WORKDIR /app

# Set environment to production
ENV NODE_ENV=production
ENV PNPM_HOME="/pnpm"

# Enable pnpm
RUN corepack enable && corepack prepare pnpm@latest --activate

# Copy package files from builder
COPY --from=builder /app/package.json ./
COPY --from=builder /app/pnpm-lock.yaml ./

# Install production dependencies only (no devDependencies)
RUN pnpm install --prod --frozen-lockfile

# Copy built application from builder stage
COPY --from=builder /app/dist ./dist

# Create non-root user for security
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nestjs -u 1001

USER nestjs

# Expose port for api-gateway
EXPOSE 3000

# # Health check
# HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
#     CMD node -e "require('http').get('http://localhost:3000/healthz', (r) => {if (r.statusCode !== 200) throw new Error(r.statusCode)})"

# Start application
CMD ["node", "dist/main.js"]
