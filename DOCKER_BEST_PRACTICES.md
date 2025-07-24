# 🐳 Containerization & Docker Best Practices

Welcome! This guide equips developers with actionable best practices for creating **optimized**, **secure**, and **portable** Docker images and managing containers efficiently. Inspired by GitHub Copilot's mission to assist with containerization excellence.

---

## 🎯 Core Principles of Containerization

### 1. Immutability
- **Principle**: Once built, container images should not be modified.
- **Best Practices**:
  - Use reproducible builds (e.g., pinned dependencies).
  - Tag images semantically (`v1.0.0`, avoid `latest` in prod).
  - Use CI/CD to automate image builds and registry pushes.
  - Treat images as versioned artifacts for rollback support.

### 2. Portability
- **Principle**: Containers should run the same anywhere (local, cloud, CI).
- **Best Practices**:
  - Externalize config using environment variables.
  - Avoid environment-specific logic in Dockerfiles.
  - Use multi-architecture base images when needed.
  - Validate config at app start-up.

### 3. Isolation
- **Principle**: Containers should run isolated from one another and the host.
- **Best Practices**:
  - Run a single process per container.
  - Avoid `host` networking; use Docker networks.
  - Use named volumes for data persistence.
  - Limit CPU/memory to ensure fair resource usage.

### 4. Efficiency & Small Images
- **Principle**: Smaller images build, ship, and run faster.
- **Best Practices**:
  - Use multi-stage builds.
  - Prefer Alpine or Distroless base images.
  - Remove unused tools/files in final image.
  - Continuously analyze and reduce image size.

---

## 🧱 Dockerfile Best Practices

### ✅ Multi-Stage Builds
- Separate build and runtime environments.
- Transfer only necessary artifacts to the final stage.

```dockerfile
FROM node:18-alpine AS build
WORKDIR /app
COPY . .
RUN npm ci && npm run build

FROM node:18-alpine
WORKDIR /app
COPY --from=build /app/dist ./dist
CMD ["node", "dist/main.js"]
```

### 🧰 Choose Minimal Base Images
Use `node:18-alpine`, `python:3.9-slim`, or `gcr.io/distroless/...`.

### 🪜 Optimize Image Layers
- Order instructions by caching efficiency.
- Combine related RUN instructions and clean up.

```dockerfile
RUN apt-get update && apt-get install -y curl \
  && apt-get clean && rm -rf /var/lib/apt/lists/*
```

### 🔍 Use .dockerignore
Exclude files like `.git`, `node_modules`, `logs`, etc.

```dockerignore
.git
node_modules
.env
*.log
test/
```

### 🔐 Use a Non-Root User
```dockerfile
RUN addgroup -S app && adduser -S app -G app
USER app
```

### 🎯 CMD & ENTRYPOINT
Use CMD for defaults, ENTRYPOINT for fixed binaries.

```dockerfile
ENTRYPOINT ["node"]
CMD ["dist/main.js"]
```

### ⚙️ Environment Variables
Use ENV to set defaults and override at runtime.

```dockerfile
ENV PORT=3000 NODE_ENV=production
```

---

## 🔒 Security Best Practices

- Use non-root users.
- Avoid storing secrets in images.
- Use Trivy, Hadolint, or Snyk to scan Dockerfiles/images.
- Sign images with Cosign or enable Docker Content Trust.
- Drop unnecessary Linux capabilities.
- Set a read-only root filesystem if possible.

---

## 🏃 Runtime Best Practices

- Set resource limits (`--memory`, `--cpus`).
- Use HEALTHCHECK for container monitoring:

```dockerfile
HEALTHCHECK CMD curl --fail http://localhost:3000/health || exit 1
```

- Centralize logs with STDOUT/STDERR.
- Use named volumes for persistence.
- Use Docker networks for service-to-service communication.

---

## 🔧 Troubleshooting Tips

| Issue | Solution |
|-------|----------|
| Large image | Use multi-stage builds, smaller base |
| Slow builds | Optimize caching, use .dockerignore |
| Container crash | Check CMD, logs, missing deps |
| Permission errors | Verify user access, volume perms |
| Network issues | Check EXPOSE, Docker network settings |

---

## ✅ Dockerfile Review Checklist

- [ ] Multi-stage builds used where appropriate
- [ ] Minimal base image with pinned version
- [ ] Layers optimized (cleaned, combined)
- [ ] .dockerignore in place
- [ ] Non-root USER defined
- [ ] Expose only required ports
- [ ] No secrets or sensitive data
- [ ] Healthcheck defined
- [ ] Static analysis integrated (Trivy, Hadolint)

---

## 📚 Recommended Tools

| Tool | Purpose |
|------|---------|
| Trivy | Scan images for vulnerabilities |
| Hadolint | Lint Dockerfiles |
| Snyk | Security scanning |
| Cosign | Image signing |
| docker-slim | Image minimization |

---

## 🚀 Example: Next.js Application Dockerfile

Here's an optimized Dockerfile for a Next.js application like Cassia Recipes:

```dockerfile
# Multi-stage build for Next.js application
FROM node:18-alpine AS base

# Install dependencies only when needed
FROM base AS deps
# Check https://github.com/nodejs/docker-node/tree/b4117f9333da4138b03a546ec926ef50a31506c3#nodealpine to understand why libc6-compat might be needed.
RUN apk add --no-cache libc6-compat
WORKDIR /app

# Install dependencies based on the preferred package manager
COPY package.json yarn.lock* package-lock.json* pnpm-lock.yaml* ./
RUN \
  if [ -f yarn.lock ]; then yarn --frozen-lockfile; \
  elif [ -f package-lock.json ]; then npm ci; \
  elif [ -f pnpm-lock.yaml ]; then yarn global add pnpm && pnpm i --frozen-lockfile; \
  else echo "Lockfile not found." && exit 1; \
  fi

# Rebuild the source code only when needed
FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Next.js collects completely anonymous telemetry data about general usage.
# Learn more here: https://nextjs.org/telemetry
# Uncomment the following line in case you want to disable telemetry during the build.
ENV NEXT_TELEMETRY_DISABLED 1

RUN npm run build

# Production image, copy all the files and run next
FROM base AS runner
WORKDIR /app

ENV NODE_ENV production
ENV NEXT_TELEMETRY_DISABLED 1

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public

# Set the correct permission for prerender cache
RUN mkdir .next
RUN chown nextjs:nodejs .next

# Automatically leverage output traces to reduce image size
# https://nextjs.org/docs/advanced-features/output-file-tracing
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs

EXPOSE 3000

ENV PORT 3000
ENV HOSTNAME "0.0.0.0"

# server.js is created by next build from the standalone output
# https://nextjs.org/docs/pages/api-reference/next-config-js/output
CMD ["node", "server.js"]
```

## 🐳 Docker Compose Example

For development with Sanity CMS:

```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - NEXT_PUBLIC_SANITY_PROJECT_ID=${NEXT_PUBLIC_SANITY_PROJECT_ID}
      - NEXT_PUBLIC_SANITY_DATASET=${NEXT_PUBLIC_SANITY_DATASET}
    volumes:
      - ./public:/app/public:ro
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - app
    restart: unless-stopped
```

---

This guide provides a comprehensive foundation for containerizing applications with Docker while following industry best practices for security, performance, and maintainability.
