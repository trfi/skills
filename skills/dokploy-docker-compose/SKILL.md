---
name: dokploy-docker-compose
description: >
  Comprehensive guide for deploying Docker Compose stacks on Dokploy (self-hosted VPS deployment platform).
  Use this skill whenever the user mentions Dokploy, deploying Docker Compose to a VPS, deploying applications
  with docker-compose.yml on a self-hosted server, or when troubleshooting Docker Compose stacks running on
  Dokploy. Covers both Compose (stack) and Application (Git) deployment types, volume strategies,
  networking, environment variables, init containers, healthchecks, and troubleshooting common errors
  like EAI_AGAIN, permission denied, unhealthy containers, and JWT/config issues. Also applies when
  deploying specific apps like LibreChat, SigNoz, or any multi-service Docker Compose stack to Dokploy.
---

# Dokploy Docker Compose Deployment Skill

## Overview

Dokploy is a self-hosted deployment platform that manages Docker containers on a VPS. It has two primary
service types and requires specific patterns to work reliably. Always read this skill before generating
any Docker Compose configurations for Dokploy.

**Quick decision tree:**
- Single-repo app with `Dockerfile`? → **Application (Git)** deployment
- Pre-built images or multi-service stack? → **Compose** deployment
- Need to mount config files without SSH? → **File Mounts** in Advanced tab
- Complex stack needing generated configs + binary downloads? → **Init Containers** pattern

---

## Part 1: Service Types

### Compose (Stack) Deployment
Best for: multi-service stacks (databases, caches, search engines, etc.) using pre-built images.

Key constraints vs local Docker Compose:
- **No relative bind mounts** (e.g., `./config.yaml`) — Dokploy's build directory is ephemeral
- Relative paths like `- type: bind, source: ./.env` will fail or mount empty directories
- Use **named volumes** and **File Mounts** instead

### Application (Git) Deployment
Best for: source code you build from a Dockerfile.

Steps: Create Service → Application → Source: Git → Build Type: Dockerfile → configure Volumes and Ports in Advanced.

---

## Part 2: Volume Strategy

This is the most critical decision when adapting any docker-compose.yml for Dokploy.

### Named Volumes (Recommended for persistent data)
```yaml
volumes:
  myapp-data:        # declared at bottom

services:
  app:
    volumes:
      - myapp-data:/app/data   # named volume
```
Docker manages these. Data persists across redeploys. Safe for databases, uploads, logs.

### File Mounts (Dokploy UI feature — for config files)
In **Advanced → Volumes → Add Volume → File Mount**:
- **Content**: paste the file text
- **File Path**: just the filename (e.g., `config.yaml`)
- **Mount Path**: full container path (e.g., `/app/config/config.yaml`)

✅ Best for: small config files (nginx.conf, app config YAML, librechat.yaml)
❌ Not for: binary files, files > a few KB, or files needing runtime generation

### Bind Mounts (VPS Host Path)
```yaml
volumes:
  - /opt/myapp/data:/app/data   # absolute host path
```
Requires SSH to create the directory first. Use for large files or files managed outside Dokploy.
Switch to "Bind Mount" tab in Dokploy's Volume UI.

### Init Container Pattern (for complex stacks)
When a stack needs multiple config files, binary downloads, or runtime-generated content, use an
alpine init container that writes files into a shared named volume:

```yaml
services:
  my-conf:          # init container
    image: alpine:3.20
    command:
      - /bin/sh
      - -lc
      - |
        cat > /config/app.yaml <<'YAML'
        key: value
        YAML
        
        # Download binaries if needed
        apk add --no-cache wget tar
        wget -O /scripts/tool.tar.gz "https://github.com/..."
        tar -xzf /tmp/tool.tar.gz -C /scripts/
    volumes:
      - app-config:/config
      - app-scripts:/scripts
    restart: "no"   # IMPORTANT: exits after running

  app:
    depends_on:
      my-conf:
        condition: service_completed_successfully   # waits for init to finish
    volumes:
      - app-config:/etc/app/
      - app-scripts:/usr/local/scripts/
```

**Shell escaping in init containers:** Use `$$` to escape `$` in heredocs so Docker doesn't
interpolate them as environment variables:
```bash
node_os=$$(uname -s | tr '[:upper:]' '[:lower:]')
```

See `references/init-container-patterns.md` for complete examples.

---

## Part 3: Networking & DNS

### The EAI_AGAIN / DNS Resolution Problem

The most common Dokploy error for multi-service stacks:
```
getaddrinfo EAI_AGAIN mongodb
```

This means a service can't resolve the hostname of another service. Causes:
1. Container started before dependency was ready (race condition)
2. Docker DNS confusion between service name vs container name

**Fix 1: Use `depends_on` with healthchecks** (prevents race conditions)
```yaml
services:
  api:
    depends_on:
      mongodb:
        condition: service_healthy   # not just service_started

  mongodb:
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      retries: 5
      start_period: 10s
```

**Fix 2: Use explicit custom network** (guarantees service-name DNS)
```yaml
services:
  api:
    networks:
      - app-net
  mongodb:
    networks:
      - app-net

networks:
  app-net:
    driver: bridge
```

**Fix 3: Use container_name in connection strings**
If service name `mongodb` doesn't resolve, try the `container_name: chat-mongodb` instead:
```
MONGO_URI=mongodb://chat-mongodb:27017/dbname
```

Docker DNS resolves `container_name` more reliably than service name in some Dokploy configurations.

---

## Part 4: Environment Variables

### Injecting env vars into Compose services
In Dokploy's **Environment tab**, set key=value pairs. In your compose YAML, reference them:

```yaml
services:
  api:
    environment:
      - JWT_SECRET=${JWT_SECRET}          # pulls from Dokploy env tab
      - MONGO_URI=${MONGO_URI}
      - PORT=3000                         # hardcoded
```

**Critical:** If you list `environment:` with specific variables, any variable NOT listed will
be ignored even if set in Dokploy's Environment tab. Either:
- List every variable explicitly: `- JWT_SECRET=${JWT_SECRET}`
- Or use `env_file` with a `.env` file mount

### Common required secrets pattern
For apps like LibreChat that need many secrets, always include in Environment tab:
```
JWT_SECRET=<random 32+ char string>
JWT_REFRESH_SECRET=<random 32+ char string>
CREDS_KEY=<exactly 64 hex chars>
CREDS_IV=<exactly 32 hex chars>
MEILI_MASTER_KEY=<random string>
```

Generate secure values: `openssl rand -hex 32`

---

## Part 5: Healthchecks & Startup Order

Always add healthchecks to databases and slow-starting services:

```yaml
# MongoDB
healthcheck:
  test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
  interval: 10s
  timeout: 5s
  retries: 5
  start_period: 10s

# ClickHouse
healthcheck:
  test: ["CMD", "wget", "--spider", "-q", "0.0.0.0:8123/ping"]
  interval: 30s
  timeout: 5s
  retries: 3

# Zookeeper (signoz/zookeeper image — uses AdminServer API on port 8080)
healthcheck:
  test: ["CMD-SHELL", "curl -s -m 2 http://localhost:8080/commands/ruok | grep error | grep null"]
  interval: 30s
  timeout: 5s
  retries: 3

# Zookeeper (bitnami image — uses nc on port 2181)
healthcheck:
  test: ["CMD-SHELL", "echo ruok | nc -w 2 127.0.0.1 2181 | grep imok"]
  interval: 30s
  timeout: 5s
  retries: 3
```

---

## Part 6: Ports & Domains

### Port configuration
```yaml
ports:
  - "8081:8080"    # host:container — exposes to internet
```
In Dokploy UI (Advanced → Ports):
- **Published Port**: host port (external)
- **Target Port**: container port (internal)
- **Mode**: Host
- **Protocol**: TCP

### Domain mapping (for HTTPS with Traefik)
After deploy → go to service → **Domains** tab:
- Add your domain (e.g., `app.yourdomain.com`)
- **Container Port**: the internal port the app listens on
- Enable HTTPS (Let's Encrypt) if needed
- When using a domain, you typically do NOT also need a published port

---

## Part 7: Updating Deployed Services

### Manual update (for stacks using public repos you don't own)
1. Go to Dokploy Dashboard → your service
2. Click **Deploy** button
3. If nothing changed (Docker cache): look for "Redeploy without Cache" or clear Build Cache in System Settings

### Automatic updates (requires owning the repo)
1. Fork the repository
2. Update service to point to your fork
3. Copy Webhook URL from Deployments tab
4. Add webhook to GitHub repo Settings → Webhooks
5. Future pushes to your fork trigger auto-deploy

---

## Part 8: Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `getaddrinfo EAI_AGAIN <hostname>` | DNS resolution failure / race condition | Add healthcheck + `depends_on: condition: service_healthy`; use container_name in URI; add custom network |
| `JwtStrategy requires a secret or key` | JWT env var not set or not passed to container | Add `JWT_SECRET=${JWT_SECRET}` to environment block; Redeploy (not just Restart) |
| `ENOENT: no such file or directory, open '/app/config.yaml'` | Config file not mounted | Use File Mount in Advanced → Volumes, or init container pattern |
| `permission denied` on volumes | Old volume data owned by different user | Rename volume in YAML (e.g., `data-v2`) to force fresh creation; or `docker volume rm` via SSH |
| Container `is unhealthy` | Healthcheck command wrong for image | Check which healthcheck method the image supports; see Part 5 |
| `cannot create agent without orgId` (SigNoz) | Normal on first startup before org is created | Access UI → complete org setup → error disappears automatically |
| `Config file YAML format is invalid: ENOENT` (LibreChat) | librechat.yaml not mounted | Create via SSH + bind mount, or File Mount in Dokploy Advanced |

---

## Part 9: Deployment SOPs

For detailed step-by-step guides, see:
- `references/sop-compose-deployment.md` — deploying any multi-service Compose stack
- `references/sop-git-deployment.md` — deploying from Git with Dockerfile
- `references/librechat-example.md` — LibreChat specific patterns (JWT, MongoDB networking, librechat.yaml)
- `references/signoz-example.md` — SigNoz specific patterns (init containers, ClickHouse, Zookeeper migration)
- `references/init-container-patterns.md` — init container patterns with heredocs and binary downloads

---

## Quick Reference: Converted docker-compose.yml Checklist

When adapting any docker-compose.yml for Dokploy:

- [ ] Replace all relative bind mounts (`./data`) with named volumes
- [ ] Remove `user: "${UID}:${GID}"` lines (causes permission errors in Dokploy)
- [ ] Move secrets from `.env` file to Dokploy Environment tab
- [ ] Reference env vars as `${VAR_NAME}` in compose YAML
- [ ] Add healthchecks to all database/dependency services
- [ ] Change `depends_on: service` to `depends_on: service: condition: service_healthy`
- [ ] For config files: use File Mounts or init container pattern
- [ ] Add custom network if experiencing DNS issues
- [ ] Use absolute paths (`/opt/myapp/`) not relative paths (`./`) for bind mounts
- [ ] Declare all named volumes at the bottom of the file
