# Dockerfile for prisma-migrate-job
# Runs Prisma migrations as a one-time job

FROM node:20-alpine

WORKDIR /app

ENV NODE_ENV=production
ENV PNPM_HOME="/pnpm"

# Enable pnpm
RUN corepack enable && corepack prepare pnpm@latest --activate

# Copy package files
COPY package.json pnpm-lock.yaml ./

# Install dependencies (including devDependencies for prisma CLI)
RUN pnpm install --frozen-lockfile

# Copy Prisma schema and migrations
COPY prisma ./prisma

# Copy built app (optional, for any helpers)
COPY dist ./dist

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nestjs -u 1001

USER nestjs

# Default command: run Prisma migration
# Can be overridden when running container
CMD ["pnpm", "prisma", "migrate", "deploy"]
