# SOP: Compose (Stack) Deployment on Dokploy

**Use this for:** Multi-service applications using pre-built Docker images.
**Examples:** LibreChat, SigNoz, Wordpress+MySQL, Grafana+Prometheus, etc.

---

## Phase 1: Prepare Your docker-compose.yml

Before pasting into Dokploy, apply these transformations to any docker-compose.yml:

### 1. Replace relative bind mounts with named volumes

Before (will fail in Dokploy):
```yaml
volumes:
  - ./data:/app/data
  - ./logs:/app/logs
```

After:
```yaml
volumes:
  - myapp-data:/app/data
  - myapp-logs:/app/logs
```

And at the bottom of the file:
```yaml
volumes:
  myapp-data:
  myapp-logs:
```

### 2. Remove user directives
```yaml
# Remove this line — causes EACCES permission errors in Dokploy
user: "${UID}:${GID}"
```

### 3. Extract secrets to Dokploy's Environment tab

Before:
```yaml
environment:
  - MONGO_URI=mongodb://admin:secret123@mongodb:27017/db
```

After (in docker-compose.yml):
```yaml
environment:
  - MONGO_URI=${MONGO_URI}
```

Then in Dokploy → Environment tab, add:
```
MONGO_URI=mongodb://admin:secret123@mongodb:27017/db
```

### 4. Remove `.env` file references

Before:
```yaml
env_file:
  - ./.env
```

After: Delete this block. Add all variables to Dokploy's Environment tab instead.

### 5. Handle config file mounts

Before:
```yaml
volumes:
  - ./config.yaml:/app/config.yaml
```

After (two options):

**Option A — File Mount (small text configs, recommended):**
Delete the bind mount line. Go to Advanced → Volumes → File Mount:
- Content: [paste file content]
- File Path: `config.yaml`
- Mount Path: `/app/config.yaml`

**Option B — Init Container (complex/multiple configs or binaries):**
See `init-container-patterns.md`

---

## Phase 2: Create the Service in Dokploy

1. Navigate to your Project in Dokploy
2. Click **Create Service** → Select **Compose**
3. Name it (e.g., `my-app-stack`)
4. Go to **Editor** tab
5. Paste your adapted YAML
6. Click **Save**

---

## Phase 3: Configure Environment Variables

1. Go to **Environment** tab of the stack
2. Add all required variables (one per line, `KEY=VALUE` format)
3. Click **Save**

---

## Phase 4: Configure Volumes (if using File Mounts)

1. Go to **Advanced** tab
2. Scroll to **Volumes/Mounts** section
3. For each config file: Click **Add Volume** → **File Mount** tab
4. Fill in Content, File Path, and Mount Path
5. Click **Create**

---

## Phase 5: Deploy

1. Click the **Deploy** button (top right)
2. Watch **Logs** tab for errors
3. Common startup errors to watch for:
   - `is unhealthy` → check dependency service logs
   - `ENOENT` for config files → file mount not attached correctly
   - `EAI_AGAIN` → DNS issue, see networking section in SKILL.md

---

## Phase 6: Configure Domain (Optional)

For web apps that need a public URL:
1. Find the web service in the container list
2. Go to **Domains** tab
3. Click **Add Domain**
4. Enter your domain (e.g., `app.yourdomain.com`)
5. **Container Port**: the app's internal listening port (e.g., `3000`, `8080`)
6. Enable HTTPS if you have DNS configured
7. Click **Create**

---

## Phase 7: Verify

- Check all containers show "Running" (green) status
- Access the application URL
- Check application-level logs for any remaining errors

---

## Maintenance

### Updating the stack
Click **Deploy** again → Dokploy pulls latest images and recreates containers.
Named volumes preserve all data automatically.

### Editing configuration
- For env vars: Edit Environment tab → **Redeploy** (not just Restart — env vars require full redeploy)
- For compose YAML: Edit in Editor tab → **Deploy**
- For File Mounts: Edit the mount content → **Deploy**

### Troubleshooting a stuck/crashed container
In Dokploy terminal or SSH:
```bash
# Restart single container without full redeploy
docker restart <container_name>

# View real-time logs
docker logs -f <container_name>

# Check container details
docker inspect <container_name>
```
