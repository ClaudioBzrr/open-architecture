# Docker & CI/CD Conventions

Patterns for containerizing full-stack applications and automating deployments.

---

## Docker Compose Structure

Use a **three-file composition** to separate shared, dev, and prod concerns:

| File | Purpose | Loaded by |
|---|---|---|
| `docker-compose.yml` | Base — services, networks, volumes | Always |
| `docker-compose.override.yml` | Dev — hot reload, bind mounts, exposed ports | `docker compose up` (automatic) |
| `docker-compose.prod.yml` | Prod — Traefik, TLS, tuned DB, backups | `-f` flag (explicit) |

### Commands

```sh
# Development (auto-loads override)
docker compose up

# Production
docker compose -f docker-compose.yml -f docker-compose.prod.yml --env-file .env up -d
```

### Base File (`docker-compose.yml`)

Defines the three core services with no environment-specific overrides:

```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks: [app-network]

  server:
    build:
      context: ./server
      dockerfile: dockerfile
    extra_hosts:
      - "host.docker.internal:host-gateway"
    environment:
      - SERVER_PORT=${SERVER_PORT}
      - DATABASE_URL=...           # or construct from DB_* vars
      # ... other env vars from .env
    networks: [app-network]
    depends_on: [db]

  web:
    build:
      context: ./web
      dockerfile: dockerfile
      args:
        - VITE_API_URL=${VITE_API_URL}    # build-time arg for frontend
    networks: [app-network]

volumes:
  postgres_data:

networks:
  app-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.19.0.0/16   # fixed subnet avoids conflicts
```

**Key decisions:**
- `extra_hosts: host.docker.internal:host-gateway` — allows the server to reach services on the host machine (e.g., external APIs that only accept localhost).
- Fixed subnet on the bridge network prevents IP conflicts across projects.

### Dev Override (`docker-compose.override.yml`)

```yaml
services:
  db:
    ports: ["5432:5432"]          # expose for local tools (DBeaver, Prisma Studio)
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER} -d ${DB_NAME}"]
      interval: 5s
      retries: 10

  server:
    build:
      target: development         # multi-stage: use dev stage
    command: npx tsx watch src/app.ts   # hot reload
    volumes:
      - ./server:/app             # bind mount source
      - /app/node_modules         # anonymous volume preserves container binaries
    environment:
      - DATABASE_URL=postgresql://${DB_USER}:${DB_PASSWORD}@db:5432/${DB_NAME}
      - CHOKIDAR_USEPOLLING=1     # required for fs watch in Docker volumes
    ports: ["${SERVER_PORT:-3000}:${SERVER_PORT:-3000}"]
    depends_on:
      db: { condition: service_healthy }   # waits for DB readiness

  web:
    build:
      target: development
    volumes:
      - ./web:/app
      - /app/node_modules
    ports: ["5173:5173"]
```

**Critical detail:** The anonymous volume `/app/node_modules` overlays the bind-mounted folder, preserving native binaries compiled for the container architecture. Without this, `node_modules` from the host would shadow the container's.

### Prod Override (`docker-compose.prod.yml`)

```yaml
services:
  traefik:
    image: traefik:v2.11
    restart: always
    ports: ["80:80", "443:443", "3001:3001"]
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /etc/traefik/:/data/
    command:
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --entrypoints.web.address=:80
      - --entrypoints.web.http.redirections.entryPoint.to=websecure
      - --entrypoints.websecure.address=:443
      - --entrypoints.apihttp.address=:3001      # API endpoint without TLS (e.g., for IP-whitelisted routes)
      - --certificatesresolvers.le.acme.tlschallenge=true
      - --certificatesresolvers.le.acme.email=${ADMIN_EMAIL}
      - --certificatesresolvers.le.acme.storage=/data/acme.json

  db:
    restart: always
    shm_size: 256mb               # must be >= shared_buffers
    command:
      - postgres
      - -c shared_buffers=256MB           # ~25% of RAM
      - -c effective_cache_size=768MB     # ~75% of RAM
      - -c work_mem=16MB
      - -c wal_compression=on
      - -c max_connections=100
      - -c log_min_duration_statement=3000  # log slow queries

  server:
    restart: always
    build: { target: production }
    volumes: ["~/server-generated/:/data/generated/"]
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.server.rule=PathPrefix(`/`)"
      - "traefik.http.routers.server.entrypoints=apihttp"
      - "traefik.http.services.server.loadbalancer.server.port=${SERVER_PORT}"

  web:
    restart: always
    build:
      target: production-stage
      args: [VITE_API_URL=/api]     # different from dev!
    environment:
      - SERVER_UPSTREAM=http://server-container:${SERVER_PORT}
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.web.rule=Host(`app.example.com`)"
      - "traefik.http.routers.web.entrypoints=websecure"
      - "traefik.http.routers.web.tls.certresolver=le"
      - "traefik.http.services.web.loadbalancer.server.port=80"

  db-backup:
    image: prodrigestivill/postgres-backup-local:16-alpine
    restart: always
    environment:
      - POSTGRES_HOST=db
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      - POSTGRES_DB=${DB_NAME}
      - SCHEDULE=@daily
      - BACKUP_KEEP_DAYS=7
    volumes: [postgres_backups:/backups]
    networks: [app-network]
    depends_on: [db]
```

**Routing in production:**
- `:443` → frontend (TLS via Let's Encrypt)
- `:3001` → server (no TLS — IP whitelist protects specific endpoints)
- Nginx in the frontend container proxies `/api/*` to the backend (eliminates CORS)

---

## Multi-Stage Dockerfiles

### Server Dockerfile

Three stages sharing a common base:

```dockerfile
# ── Base: install deps + generate Prisma client ──────────────────────────────
FROM node:24-alpine AS base
WORKDIR /app
COPY package*.json ./
COPY prisma ./prisma
RUN npm ci
ENV DATABASE_URL=postgresql://placeholder:placeholder@localhost:5432/placeholder
RUN npx prisma generate           # validates schema at build time
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# ── Development: hot-reload ──────────────────────────────────────────────────
FROM base AS development
COPY . .
RUN mkdir -p /data/generated
ENTRYPOINT ["/entrypoint.sh"]
CMD ["node", "--import", "tsx", "--watch", "src/app.ts"]

# ── Production: direct execution ─────────────────────────────────────────────
FROM base AS production
COPY . .
RUN mkdir -p /data/generated
ENTRYPOINT ["/entrypoint.sh"]
CMD ["npm", "start"]
```

**Key points:**
- `prisma generate` runs at build time with a placeholder URL — validates the schema without connecting.
- The real `DATABASE_URL` is injected at runtime via docker-compose.
- Non-root user (`appuser`) for security.

### Web Dockerfile

Three stages: dev → build → production (nginx):

```dockerfile
# ── Development: Vite HMR ────────────────────────────────────────────────────
FROM node:24-alpine AS development
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
EXPOSE 5173
CMD ["npm", "run", "dev", "--", "--host"]

# ── Build: compile static assets ─────────────────────────────────────────────
FROM node:24-alpine AS build-stage
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
ARG VITE_API_URL
ENV VITE_API_URL=$VITE_API_URL     # baked into the JS bundle
RUN npm run build

# ── Production: serve via nginx ──────────────────────────────────────────────
FROM nginx:alpine AS production-stage
COPY --from=build-stage /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf.template
ENV SERVER_UPSTREAM=http://server:3000
EXPOSE 80
CMD ["/bin/sh", "-c", "envsubst '$SERVER_UPSTREAM' < /etc/nginx/conf.d/default.conf.template > /etc/nginx/conf.d/default.conf && nginx -g 'daemon off;'"]
```

**Key points:**
- `VITE_API_URL` is a **build-time** variable — it gets baked into the JS bundle. Changing it requires a rebuild.
- Production stage uses nginx with `envsubst` to inject `SERVER_UPSTREAM` at runtime.
- The production build target differs from dev (`production-stage` vs `development`).

---

## Entrypoint Pattern

The server's `entrypoint.sh` runs as root, then drops privileges:

```sh
#!/bin/sh
set -e

# 1. Fix volume ownership (runs as root)
chown -R appuser:appgroup /data/generated 2>/dev/null || true

# 2. Construct DATABASE_URL with percent-encoded credentials
DATABASE_URL=$(node -e "
  const u = encodeURIComponent(process.env.DB_USER);
  const p = encodeURIComponent(process.env.DB_PASSWORD);
  const n = process.env.DB_NAME;
  process.stdout.write('postgresql://' + u + ':' + p + '@db:5432/' + n);
")
export DATABASE_URL

# 3. Run migrations + regenerate client
npx prisma migrate deploy
npx prisma generate

# 4. Drop to non-root user
exec su-exec appuser "$@"
```

**Why this pattern:**
- `chown` must run as root to fix mounted volume permissions.
- `prisma migrate deploy` + `prisma generate` ensures the schema is current on every boot.
- `exec su-exec appuser "$@"` replaces the shell process with the app under a non-root user.

---

## Nginx Configuration

For SPA frontends that proxy API calls to a backend:

```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    # Trust reverse proxy for real IP extraction
    set_real_ip_from 172.16.0.0/12;
    real_ip_header X-Forwarded-For;

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_types text/plain text/css application/javascript application/json image/svg+xml;

    # Cache hashed assets (Vite content hashes) aggressively
    location ~* \.(?:js|css|woff2?|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # Docker DNS resolver for dynamic upstream resolution
    resolver 127.0.0.11 valid=10s ipv6=off;

    # Proxy /api/* to backend (eliminates CORS)
    location /api/ {
        set $api_upstream "${SERVER_UPSTREAM}";
        rewrite ^/api/(.*) /$1 break;
        proxy_pass $api_upstream;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # SPA fallback — always return index.html
    location / {
        add_header Cache-Control "no-cache";
        add_header X-Frame-Options "DENY";
        add_header X-Content-Type-Options "nosniff";
        try_files $uri $uri/ /index.html;
    }
}
```

**Key decisions:**
- `resolver 127.0.0.11` — Docker's internal DNS. Enables nginx to re-resolve upstream hostnames if the backend container restarts.
- `envsubst` injects `SERVER_UPSTREAM` at container start, not at build time.
- Hashed assets (JS/CSS) get `immutable` + 1-year cache. HTML gets `no-cache`.

---

## CI/CD (GitHub Actions)

### Pattern: Push-to-Deploy via SCP + SSH

```yaml
on:
  push:
    branches: [master]

name: ci
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Get latest code
        uses: actions/checkout@v3

      - name: Copy files to VPS
        uses: appleboy/scp-action@v1.0.0
        with:
          host: ${{ secrets.HOST }}
          username: ${{ secrets.USERNAME }}
          key: ${{ secrets.PRIVATE_KEY }}
          port: ${{ secrets.SSH_PORT }}
          source: "."
          target: "~/dev/project-name"

      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.HOST }}
          username: ${{ secrets.USERNAME }}
          key: ${{ secrets.PRIVATE_KEY }}
          port: ${{ secrets.SSH_PORT }}
          script: |
            cd ~/dev/project-name

            # Write .env from GitHub secrets (single-quoted to prevent shell expansion)
            {
              printf 'DATABASE_URL=%s\n'  '${{ secrets.DATABASE_URL }}'
              printf 'JWT_SECRET=%s\n'     '${{ secrets.JWT_SECRET }}'
              # ... other secrets
            } > .env

            # Deploy
            docker compose -f docker-compose.yml -f docker-compose.prod.yml \
              --env-file .env up -d --build --force-recreate --remove-orphans

            # Clean up — .env must not remain on the server
            rm .env
```

**Key decisions:**
- Secrets are resolved by GitHub Actions **before** the SSH script is sent to the server.
- Values are single-quoted in the `printf` to prevent shell expansion on the remote host.
- `.env` is created, used, and deleted in the same step — never persists on disk.
- `--force-recreate --remove-orphans` ensures clean state after deploy.

### Secrets Required

| Secret | Purpose |
|---|---|
| `HOST` | VPS IP address |
| `SSH_PORT` | SSH port (usually 22) |
| `USERNAME` | SSH user |
| `PRIVATE_KEY` | SSH private key for authentication |
| `DATABASE_URL` | Full PostgreSQL connection string |
| `JWT_SECRET` | Token signing secret |
| `DB_USER`, `DB_PASSWORD`, `DB_NAME` | Database credentials |
| `*_LOGIN`, `*_PASSWORD` | External API credentials |
| `EMAIL_*` | SMTP configuration |
| `ADMIN_EMAIL` | Let's Encrypt registration |

---

## Docker Pitfalls

1. **`VITE_API_URL` is build-time** — changing it requires `docker compose build`, not just `up`.
2. **Anonymous `node_modules` volume** — without it, host binaries shadow container binaries.
3. **`shm_size` for PostgreSQL** — must be `>= shared_buffers` or Postgres may crash.
4. **`CHOKIDAR_USEPOLLING=1`** — required for file watching in Docker bind mounts (inotify doesn't work).
5. **Entrypoint runs as root** — use `su-exec` or `gosu` to drop privileges after setup.
6. **`.env` on the server** — create it, use it, delete it. Never leave it on disk.
7. **Fixed subnet** — prevents IP conflicts when running multiple projects on the same host.
8. **`host.docker.internal:host-gateway`** — needed to reach host-side services from inside containers.
