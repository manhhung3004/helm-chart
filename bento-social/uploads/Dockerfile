# Multi-stage Dockerfile for upload-service
# Optimized for file handling and storage integration

FROM node:20-alpine AS builder

WORKDIR /app

ENV PNPM_HOME="/pnpm"
RUN corepack enable && corepack prepare pnpm@latest --activate

COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile

COPY . .
RUN pnpm build

# ============================================
# Runtime Stage
# ============================================
FROM node:20-alpine AS runtime

WORKDIR /app

ENV NODE_ENV=production
ENV PNPM_HOME="/pnpm"

RUN corepack enable && corepack prepare pnpm@latest --activate

COPY --from=builder /app/package.json ./
COPY --from=builder /app/pnpm-lock.yaml ./

RUN pnpm install --prod --frozen-lockfile

COPY --from=builder /app/dist ./dist

# Create upload directory (if using local storage)
RUN mkdir -p /app/uploads && chown -R nestjs:nodejs /app/uploads

RUN addgroup -g 1001 -S nodejs && \
    adduser -S nestjs -u 1001

USER nestjs

EXPOSE 3000

# HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
#     CMD node -e "require('http').get('http://localhost:3000/healthz', (r) => {if (r.statusCode !== 200) throw new Error(r.statusCode)})"

CMD ["node", "dist/main.js"]
