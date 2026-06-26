# Docker

Use reproducible, minimal, and least-privilege Dockerfiles.

## Base images

Use specific version tags. Avoid `latest` in production Dockerfiles.

Good:

```dockerfile
FROM python:3.12-slim
```

Bad:

```dockerfile
FROM python:latest
```

## Multi-stage builds

Use multi-stage builds to keep final images small and avoid shipping build dependencies.

Good:

```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app
COPY --from=build /app/dist ./dist
USER node
CMD ["node", "dist/index.js"]
```

## User permissions

Run containers as a non-root user. Avoid running production containers as root.

Good:

```dockerfile
FROM python:3.12-slim
RUN useradd -m appuser
USER appuser
```

Bad:

```dockerfile
FROM python:latest
# Running as root
```
