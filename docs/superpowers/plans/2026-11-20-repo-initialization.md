# repo_for_lecture Initialization Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Initialize repo_for_lecture as a production-ready project with local dev environment and server infrastructure, ready for agents to develop features following the proper workflow (story → branch → tests → code → PR → Sonar → merge).

**Architecture:** 
- Local: Template renamed and git-initialized, local stack testable via `npm run up`
- Server: Self-hosted Supabase, Postgres, Next.js, SafeLine reverse-proxy with SSL
- CI/CD: GitHub Actions triggers deploy script on main branch push; SSH into deploy user, git pull, docker compose restart
- SonarQube: Community version self-hosted with branch plugin, comments on PRs via GitHub App

**Tech Stack:** TypeScript, Next.js (App Router), Supabase Edge Functions (Deno), Postgres, Docker Compose, SafeLine WAF, SonarQube Community 26.5.0 + branch plugin, GitHub Actions

**Spec:** `docs/CLAUDE.md` (project rules) + this plan

## Global Constraints

- Stack is locked: TypeScript everywhere, Next.js, self-hosted Supabase, Deno Edge Functions
- SonarQube: Community 26.5.0 with branch plugin (versions change together only)
- RLS enabled on every table, service_role key forbidden except with explicit justification
- No SQLite in production code, no mock databases except at test boundaries
- Port publishing in docker-compose only on 127.0.0.1; public traffic only through SafeLine
- Postgres in production never published to host
- No secrets (.env, tokens, deploy keys) committed to git
- One task = one branch = one PR; never reuse branches
- Merge only on green CI; never bypass or disable pipeline

## Review Focus

1. **Deploy user permissions:** Ensure deploy user can git pull, docker pull, docker compose restart without sudo
2. **SafeLine SSL:** Let's Encrypt certificates auto-renew and are valid for all four subdomains (app, api, sonar, safeline)
3. **SonarQube self-hosted Community:** Branch plugin installed and working; GitHub App integrated for PR comments
4. **Local stack startup:** `npm run up` completes without errors; services accessible on expected localhost ports
5. **CD pipeline security:** SSH key in GitHub secrets never logged; deploy script validates target host before executing

---

## Task 1: Rename Template to repo_for_lecture Locally

**Files:**
- Modify: `package.json`
- Modify: `sonar-project.properties`
- Modify: `supabase/config.toml`
- Modify: `.github/workflows/*.yml` (if template-for-mvp referenced)
- Create: `.env.example` (if not present)

**Interfaces:**
- Consumes: (none; first task)
- Produces: Project name and IDs consistently set to `repo_for_lecture` across config files

**Steps:**

- [ ] **Step 1: Update package.json**

Open `package.json` and change the `name` field from `template-for-mvp` to `repo_for_lecture` (lowercase). Keep everything else unchanged.

- [ ] **Step 2: Update sonar-project.properties**

Open `sonar-project.properties`. Update:
- `sonar.projectKey=repo_for_lecture`
- `sonar.projectName=repo_for_lecture`

- [ ] **Step 3: Update supabase/config.toml**

Open `supabase/config.toml`. Update:
- `project_id="repo_for_lecture"`

- [ ] **Step 4: Check .github/workflows for template references**

Grep all `.yml` files in `.github/workflows/` for `template-for-mvp`. If found in comments or naming that affects behavior, update them. Keep workflow names descriptive (e.g., `ci.yml`, `deploy.yml`).

```bash
grep -r "template-for-mvp" .github/workflows/
```

- [ ] **Step 5: Verify .env.example exists**

Check that `.env.example` and `infra/.env.example` exist (without real values). If missing, create them with placeholder variable names (not from actual `.env`).

- [ ] **Step 6: Commit**

```bash
git init
git add package.json sonar-project.properties supabase/config.toml .github/workflows/ .env.example infra/.env.example
git commit -m "chore: rename template-for-mvp to repo_for_lecture"
```

**Review test for Constraint:** Grep project name in key files:
```bash
grep -E "repo_for_lecture|template-for-mvp" package.json sonar-project.properties supabase/config.toml
```
Expected: All references show `repo_for_lecture`, none show `template-for-mvp`.

---

## Task 2: Initialize Git Remote and First Push

**Files:**
- Modify: `.git/config` (git remote)
- Modify: `.gitignore` (verify secrets are excluded)

**Interfaces:**
- Consumes: (from Task 1: renamed project structure)
- Produces: Git remote set to `git@github.com:andreydorozhkin/repo_for_lecture.git`; first commit pushed to main

**Steps:**

- [ ] **Step 1: Verify git status before remote setup**

```bash
git status
```
Expected: No uncommitted changes (all from Task 1 committed).

- [ ] **Step 2: Set git remote**

```bash
git remote add origin git@github.com:andreydorozhkin/repo_for_lecture.git
git branch -M main
```

- [ ] **Step 3: Verify SSH key works**

```bash
ssh -T git@github.com
```
Expected: "Hi andreydorozhkin! You've successfully authenticated..." (proves SSH key is set up in GitHub).

- [ ] **Step 4: Verify .gitignore excludes secrets**

Open `.gitignore`. Ensure it contains:
```
.env
.env.local
infra/.env
infra/.env.prod
infra/safeline/.env
app/.env.local
*.pem
```

If not present, add them.

- [ ] **Step 5: Run pre-push checks**

Before pushing, run:
```bash
npm run lint
npm run typecheck
npm test
npm run test:functions
```

Expected: All pass (green).

- [ ] **Step 6: Scan git history for secrets with gitleaks**

```bash
docker run --rm -v "$PWD":/repo -w /repo ghcr.io/gitleaks/gitleaks:v8.30.1 git --config .gitleaks.toml --redact --no-banner
```

Expected: No secrets found.

- [ ] **Step 7: Push to main**

```bash
git push -u origin main
```

Expected: Push succeeds; remote shows commit on main branch.

**Review test for Constraint:** Verify no secrets in git:
```bash
git log --all -p | grep -iE "password|token|secret|key" | head -5
```
Expected: No matches (or only comments about avoiding secrets).

---

## Task 3: Verify Local Stack with npm run up

**Files:**
- Modify: (none; just verification)

**Interfaces:**
- Consumes: (from Task 1: initialized project)
- Produces: Confirmation that local docker-compose and npm scripts work; services accessible on localhost ports

**Steps:**

- [ ] **Step 1: Ensure Docker is running locally**

```bash
docker --version
docker compose version
```

Expected: Both return version numbers (not "command not found").

- [ ] **Step 2: Run npm run up**

```bash
npm run up
```

Expected: Docker containers start; logs show Supabase, Postgres, Next.js services starting without fatal errors. Wait ~30 seconds for Supabase to fully initialize.

- [ ] **Step 3: Verify localhost ports are accessible**

In another terminal, check:
```bash
curl -s http://localhost:3000/ | head -5  # Next.js
curl -s http://localhost:5432 || echo "Postgres port open"
curl -s http://localhost:8000/rest/v1/ | head -5  # Supabase API
```

Expected: Responses indicate services are running (200 OK or Postgres connection accepted).

- [ ] **Step 4: Check Supabase dashboard**

Open `http://localhost:5555` in browser. Expected: Supabase Studio loads, showing project, tables, auth, Edge Functions menus.

- [ ] **Step 5: Verify migrations ran**

In Supabase Studio, navigate to SQL Editor. Run:
```sql
SELECT * FROM information_schema.tables WHERE table_schema='public';
```

Expected: Tables listed (at minimum, should see auth-related tables if Supabase is healthy). If migrations are empty, that's OK; schema is defined in `supabase/migrations/`.

- [ ] **Step 6: Stop the stack**

```bash
npm run down
```

Expected: Containers stop cleanly; no errors in logs.

**Review test for Constraint:** Confirm RLS is set on tables (if schema exists):
```bash
docker exec <postgres-container> psql -U postgres -d postgres -c "SELECT table_name, row_level_security FROM information_schema.tables WHERE table_schema='public';"
```
Expected: RLS is ON for all application tables (not system tables).

---

## Task 4: Create Deploy User on Production Server

**Files:**
- Create: (remote) `/home/deploy/.ssh/authorized_keys`
- Create: (local for reference) `.github/deploy/ssh-setup.sh`

**Interfaces:**
- Consumes: (SSH access to 72.56.67.176 as root)
- Produces: Deploy user `deploy` with passwordless SSH key access and sudo rights for docker commands only

**Steps:**

- [ ] **Step 1: SSH to server and create deploy user**

```bash
ssh root@72.56.67.176

# On server:
useradd -m -s /bin/bash deploy
usermod -aG docker deploy
mkdir -p /home/deploy/.ssh
chmod 700 /home/deploy/.ssh
```

- [ ] **Step 2: Generate or use existing deploy SSH key**

If you don't have a dedicated deploy key, generate one locally (do NOT commit):

```bash
ssh-keygen -t ed25519 -f ~/.ssh/github_deploy_key -N "" -C "deploy@repo_for_lecture"
cat ~/.ssh/github_deploy_key.pub
```

Copy the public key output.

- [ ] **Step 3: Add public key to deploy user on server**

```bash
# Back on server as root:
echo "<paste-public-key-here>" >> /home/deploy/.ssh/authorized_keys
chmod 600 /home/deploy/.ssh/authorized_keys
chown deploy:deploy /home/deploy/.ssh/authorized_keys
```

- [ ] **Step 4: Configure sudo for docker commands without password**

```bash
# On server as root:
echo "deploy ALL=(ALL) NOPASSWD: /usr/bin/docker, /usr/bin/docker-compose" >> /etc/sudoers.d/deploy
chmod 440 /etc/sudoers.d/deploy
```

- [ ] **Step 5: Verify deploy user can connect and run docker**

```bash
# Locally, test SSH login:
ssh -i ~/.ssh/github_deploy_key deploy@72.56.67.176 "docker ps"
```

Expected: Docker command runs without password prompt; container list appears (or empty list if no containers yet).

- [ ] **Step 6: Create a reference deployment script locally (not committed)**

Create `.github/deploy/ssh-setup.sh` as documentation (this file is NOT part of the commit; it's for your reference):

```bash
#!/bin/bash
# Deploy script template (reference only; real version in GitHub Actions)
set -e
DEPLOY_KEY="$1"
SERVER="72.56.67.176"
DEPLOY_USER="deploy"

mkdir -p ~/.ssh
echo "$DEPLOY_KEY" > ~/.ssh/deploy_key
chmod 600 ~/.ssh/deploy_key

ssh-keyscan -H "$SERVER" >> ~/.ssh/known_hosts 2>/dev/null || true

ssh -i ~/.ssh/deploy_key "$DEPLOY_USER@$SERVER" <<EOF
  set -e
  cd /opt/repo_for_lecture || { echo "Repository not cloned yet"; exit 1; }
  git pull origin main
  docker compose restart
EOF
```

This is for reference; the real version will be in `.github/workflows/deploy.yml`.

**Review test for Constraint:** Confirm deploy user cannot escalate privileges:

```bash
ssh -i ~/.ssh/github_deploy_key deploy@72.56.67.176 "sudo whoami"
```

Expected: Returns "root" (sudo works for authorized commands). Then verify they can't edit sudoers:

```bash
ssh -i ~/.ssh/github_deploy_key deploy@72.56.67.176 "sudo cat /etc/shadow"
```

Expected: Permission denied (can only run docker, not arbitrary root commands).

---

## Task 5: Install Docker and Set Up Server Infrastructure

**Files:**
- Create: (remote) `/opt/repo_for_lecture/docker-compose.prod.yml`
- Create: (remote) `/opt/repo_for_lecture/.env` (for server; not committed)
- Modify: (local) `infra/docker-compose.yml` (or create `infra/docker-compose.prod.yml`)
- Modify: `Dockerfile` or create server-specific variant if needed

**Interfaces:**
- Consumes: (from Task 4: deploy user ready)
- Produces: Docker and Docker Compose installed on server; repo cloned to `/opt/repo_for_lecture`; docker-compose.prod.yml ready to start services

**Steps:**

- [ ] **Step 1: Check if Docker is installed on server**

```bash
ssh root@72.56.67.176 "docker --version && docker compose version"
```

If both commands return version numbers, skip to Step 3. If not:

- [ ] **Step 2: Install Docker on server**

```bash
ssh root@72.56.67.176 <<'EOF'
apt-get update
apt-get install -y docker.io docker-compose-v2
systemctl start docker
systemctl enable docker
docker --version
EOF
```

- [ ] **Step 3: Create deployment directory and clone repo**

```bash
ssh root@72.56.67.176 <<'EOF'
mkdir -p /opt/repo_for_lecture
cd /opt/repo_for_lecture
git clone --branch main git@github.com:andreydorozhkin/repo_for_lecture.git .
EOF
```

- [ ] **Step 4: Create production docker-compose file**

Locally, check if `infra/docker-compose.prod.yml` exists. If not, copy or create it:

```bash
cp infra/docker-compose.yml infra/docker-compose.prod.yml
```

Then edit `infra/docker-compose.prod.yml`:
- Change `127.0.0.1` port bindings to only Postgres (all others go through SafeLine reverse-proxy)
- Increase resource limits (memory, CPU) if needed
- Add restart policy: `restart: unless-stopped`
- Ensure no dev-only services (e.g., adminer, local SonarQube if dev-only) are commented out or removed

Example for ports (Postgres internal only, others on specific bind):
```yaml
services:
  postgres:
    ports:
      - "127.0.0.1:5432:5432"  # Internal only
  
  supabase:
    # Port 8000 not exposed; accessed through SafeLine
    
  next:
    # Port 3000 not exposed; accessed through SafeLine
```

- [ ] **Step 5: Create production .env file on server**

SSH to server and create `/opt/repo_for_lecture/.env`:

```bash
ssh root@72.56.67.176 "cat > /opt/repo_for_lecture/.env <<'ENVEOF'
# Supabase
SUPABASE_URL=https://api.notalike.space
SUPABASE_ANON_KEY=<get-from-supabase-after-startup>
SUPABASE_SERVICE_ROLE_KEY=<get-from-supabase-after-startup>

# Next.js
NEXT_PUBLIC_SUPABASE_URL=https://api.notalike.space
NEXT_PUBLIC_SUPABASE_ANON_KEY=<same-as-above>
NODE_ENV=production

# App
APP_URL=https://app.notalike.space
API_URL=https://api.notalike.space
ENVEOF
"
```

(Keys will be populated once Supabase starts.)

- [ ] **Step 6: Set proper permissions on repo and .env**

```bash
ssh root@72.56.67.176 <<EOF
chown -R deploy:deploy /opt/repo_for_lecture
chmod 600 /opt/repo_for_lecture/.env
EOF
```

- [ ] **Step 7: Commit docker-compose.prod.yml locally**

```bash
git add infra/docker-compose.prod.yml
git commit -m "infra: add production docker-compose configuration"
git push origin main
```

Then pull on server:

```bash
ssh deploy@72.56.67.176 "cd /opt/repo_for_lecture && git pull origin main"
```

**Review test for Constraint:** Verify port bindings in docker-compose.prod.yml:

```bash
grep -E "ports:|127\.0\.0\.1" infra/docker-compose.prod.yml
```

Expected: Only Postgres has `127.0.0.1` binding; other services either have no ports section or bind to all interfaces (0.0.0.0, accessed through SafeLine).

---

## Task 6: Set Up SafeLine Reverse Proxy with Let's Encrypt SSL

**Files:**
- Create: (remote) `/opt/safeline/docker-compose.yml`
- Create: (remote) `/opt/safeline/.env`
- Create: (remote) `/opt/safeline/nginx.conf` or use SafeLine built-in routing

**Interfaces:**
- Consumes: (from Task 5: app services running internally)
- Produces: SafeLine reverse-proxy listening on 80/443, routing to app/api/sonar/safeline subdomains with valid SSL certs

**Steps:**

- [ ] **Step 1: SSH to server and create SafeLine directory**

```bash
ssh root@72.56.67.176 <<EOF
mkdir -p /opt/safeline
cd /opt/safeline
EOF
```

- [ ] **Step 2: Create SafeLine docker-compose**

Create `/opt/safeline/docker-compose.yml`:

```yaml
version: '3.8'

services:
  safeline:
    image: "chaitin/safeline:latest"
    container_name: safeline
    ports:
      - "80:80"
      - "443:443"
      - "9443:9443"  # SafeLine admin panel (optional, can restrict to 127.0.0.1)
    volumes:
      - safeline_data:/data
      - /opt/safeline/nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - certbot_certs:/etc/letsencrypt:ro
    environment:
      - SAFELINE_ADMIN_PORT=9443
    networks:
      - proxy
    restart: unless-stopped

  certbot:
    image: certbot/certbot
    container_name: certbot
    volumes:
      - certbot_certs:/etc/letsencrypt
      - certbot_www:/var/www/certbot
    entrypoint: /bin/sh -c "certbot renew --webroot -w /var/www/certbot --quiet"
    restart: unless-stopped

volumes:
  safeline_data:
  certbot_certs:
  certbot_www:

networks:
  proxy:
    driver: bridge
```

- [ ] **Step 3: Create nginx configuration for routing**

Create `/opt/safeline/nginx.conf`:

```nginx
upstream app {
    server 127.0.0.1:3000;
}

upstream api {
    server 127.0.0.1:8000;
}

upstream sonar {
    server 127.0.0.1:9000;
}

server {
    listen 80;
    server_name app.notalike.space api.notalike.space sonar.notalike.space safeline.notalike.space;

    # Let's Encrypt challenge
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    # Redirect to HTTPS
    location / {
        return 301 https://$server_name$request_uri;
    }
}

server {
    listen 443 ssl http2;
    server_name app.notalike.space;

    ssl_certificate /etc/letsencrypt/live/notalike.space/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/notalike.space/privkey.pem;

    location / {
        proxy_pass http://app;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 443 ssl http2;
    server_name api.notalike.space;

    ssl_certificate /etc/letsencrypt/live/notalike.space/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/notalike.space/privkey.pem;

    location / {
        proxy_pass http://api;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 443 ssl http2;
    server_name sonar.notalike.space;

    ssl_certificate /etc/letsencrypt/live/notalike.space/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/notalike.space/privkey.pem;

    location / {
        proxy_pass http://sonar;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 443 ssl http2;
    server_name safeline.notalike.space;

    ssl_certificate /etc/letsencrypt/live/notalike.space/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/notalike.space/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:9443;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

- [ ] **Step 4: Initialize Let's Encrypt certificates**

```bash
ssh root@72.56.67.176 <<EOF
cd /opt/safeline
docker compose up -d certbot
# Wait for certbot to issue initial certificate
sleep 30
docker compose logs certbot
EOF
```

Expected: Certbot output shows certificate issued for notalike.space and all subdomains.

- [ ] **Step 5: Start SafeLine**

```bash
ssh root@72.56.67.176 <<EOF
cd /opt/safeline
docker compose up -d
sleep 20
docker compose logs safeline
EOF
```

Expected: SafeLine container running, port 443 listening.

- [ ] **Step 6: Test SSL certificate**

```bash
openssl s_client -connect app.notalike.space:443 -servername app.notalike.space < /dev/null
```

Expected: Certificate chain shown; verify issuer is "Let's Encrypt" and CN includes notalike.space.

- [ ] **Step 7: Verify HTTPS access to subdomains**

```bash
curl -s -I https://app.notalike.space
curl -s -I https://api.notalike.space
curl -s -I https://sonar.notalike.space
```

Expected: HTTP 200 (or proxy error if services not yet running, but SSL handshake succeeds).

**Review test for Constraint:** Confirm SafeLine listens only on 80/443 (public):

```bash
ssh root@72.56.67.176 "docker inspect safeline | grep -A 10 Ports"
```

Expected: Ports show `80/tcp` and `443/tcp` mapped to 0.0.0.0, no other ports exposed externally.

---

## Task 7: Start Server Stack and Initialize Supabase

**Files:**
- Modify: (remote) `/opt/repo_for_lecture/.env` (populate Supabase keys)

**Interfaces:**
- Consumes: (from Task 5: docker-compose.prod.yml ready; from Task 6: SafeLine routing ready)
- Produces: Server services running; Supabase initialized with database schema; keys populated in .env

**Steps:**

- [ ] **Step 1: Start the server stack**

```bash
ssh deploy@72.56.67.176 <<EOF
cd /opt/repo_for_lecture
docker compose -f infra/docker-compose.prod.yml up -d
sleep 30
docker compose -f infra/docker-compose.prod.yml logs
EOF
```

Expected: Logs show Postgres starting, Supabase API initializing, Next.js ready.

- [ ] **Step 2: Verify Supabase is reachable through SafeLine**

```bash
curl -s https://api.notalike.space/rest/v1/ | head -5
```

Expected: JSON response (Supabase API is responding).

- [ ] **Step 3: Get Supabase keys from server**

SSH to server and retrieve keys:

```bash
ssh root@72.56.67.176 <<EOF
# Supabase stores default keys in its database; retrieve them:
docker exec $(docker ps -q -f "name=supabase") \
  psql -U postgres -d postgres -c \
  "SELECT api_key FROM supabase.auth_function WHERE id='anon' LIMIT 1;"
EOF
```

Or access Supabase Studio directly (if exposed) and copy keys from API tab. Alternatively, check server logs:

```bash
ssh root@72.56.67.176 "docker compose -f infra/docker-compose.prod.yml logs supabase | grep -i key | head -5"
```

**Note:** This may require accessing Supabase Studio at `http://localhost:5555` on the server or through a tunnel. For now, assume keys are auto-generated and printed in startup logs.

- [ ] **Step 4: Populate .env on server with Supabase keys**

```bash
ssh root@72.56.67.176 "cat > /opt/repo_for_lecture/.env <<'ENVEOF'
SUPABASE_URL=https://api.notalike.space
SUPABASE_ANON_KEY=<paste-anon-key-from-step-3>
SUPABASE_SERVICE_ROLE_KEY=<paste-service-role-key-from-step-3>

NEXT_PUBLIC_SUPABASE_URL=https://api.notalike.space
NEXT_PUBLIC_SUPABASE_ANON_KEY=<paste-anon-key-from-step-3>
NODE_ENV=production

APP_URL=https://app.notalike.space
API_URL=https://api.notalike.space
ENVEOF
"
```

- [ ] **Step 5: Restart services to load new .env**

```bash
ssh deploy@72.56.67.176 <<EOF
cd /opt/repo_for_lecture
docker compose -f infra/docker-compose.prod.yml restart next supabase
sleep 10
docker compose -f infra/docker-compose.prod.yml logs next supabase | tail -20
EOF
```

Expected: Services restart cleanly; no connection errors in logs.

- [ ] **Step 6: Verify app is accessible through SafeLine**

```bash
curl -s https://app.notalike.space | head -20
```

Expected: HTML response from Next.js (app home page or layout).

**Review test for Constraint:** Confirm Postgres is not exposed externally:

```bash
ssh root@72.56.67.176 "netstat -tlnp | grep 5432"
```

Expected: Shows `127.0.0.1:5432` (loopback only), not `0.0.0.0:5432`.

---

## Task 8: Set Up GitHub Secrets and Deploy SSH Key

**Files:**
- Modify: (GitHub repo settings; not local files)

**Interfaces:**
- Consumes: (from Task 4: deploy SSH private key; from Task 7: server running)
- Produces: GitHub secrets configured for CI/CD and Sonar

**Steps:**

- [ ] **Step 1: Encode deploy SSH private key for GitHub secret**

Locally, the private key is at `~/.ssh/github_deploy_key`. Encode it:

```bash
cat ~/.ssh/github_deploy_key | base64 -w 0 | pbcopy
```

(On Linux, use `base64 -w 0` without `pbcopy`; manually copy the output.)

- [ ] **Step 2: Add GitHub secrets via web UI**

Go to `https://github.com/andreydorozhkin/repo_for_lecture/settings/secrets/actions` and add:

| Secret Name | Value |
|---|---|
| `DEPLOY_KEY` | `<base64-encoded-ssh-key-from-step-1>` |
| `DEPLOY_USER` | `deploy` |
| `DEPLOY_HOST` | `72.56.67.176` |
| `DEPLOY_PATH` | `/opt/repo_for_lecture` |
| `SONAR_HOST_URL` | `https://sonar.notalike.space` |
| `SONAR_TOKEN` | `<will-generate-after-sonar-starts>` |

(Leave `SONAR_TOKEN` blank for now; populate after SonarQube is running.)

- [ ] **Step 3: Verify secrets are saved**

In GitHub repo settings, Secrets & variables → Actions, confirm all secrets appear as `•••••••••` (redacted).

**Review test for Constraint:** Confirm deploy key is not logged in GitHub Actions runs:

Later, after CD pipeline runs, check the action logs (never expose the key). The logs should show `****` where the key is used.

---

## Task 9: Create GitHub Actions CD Pipeline

**Files:**
- Create: `.github/workflows/deploy.yml`

**Interfaces:**
- Consumes: (from Task 8: GitHub secrets configured; from Task 5: deploy user ready)
- Produces: GitHub Actions workflow that triggers on push to main, deploys to server via SSH

**Steps:**

- [ ] **Step 1: Create deploy.yml workflow**

Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Decode and setup SSH key
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.DEPLOY_KEY }}" | base64 -d > ~/.ssh/deploy_key
          chmod 600 ~/.ssh/deploy_key
          ssh-keyscan -H ${{ secrets.DEPLOY_HOST }} >> ~/.ssh/known_hosts 2>/dev/null || true

      - name: Deploy to production
        run: |
          ssh -i ~/.ssh/deploy_key ${{ secrets.DEPLOY_USER }}@${{ secrets.DEPLOY_HOST }} <<'DEPLOY_EOF'
          set -e
          cd ${{ secrets.DEPLOY_PATH }}
          git pull origin main
          docker compose -f infra/docker-compose.prod.yml pull
          docker compose -f infra/docker-compose.prod.yml up -d
          docker compose -f infra/docker-compose.prod.yml logs --tail 20
          DEPLOY_EOF

      - name: Verify deployment
        run: |
          sleep 5
          curl -s -I https://app.notalike.space | grep -q "HTTP" && echo "✓ App is up" || echo "✗ App is down"
```

- [ ] **Step 2: Commit and push the workflow**

```bash
git add .github/workflows/deploy.yml
git commit -m "ci: add deployment pipeline for main branch"
git push origin main
```

- [ ] **Step 3: Monitor the GitHub Actions run**

Go to `https://github.com/andreydorozhkin/repo_for_lecture/actions` and watch the workflow run. Expected: Workflow completes successfully; app is reachable.

**Review test for Constraint:** Verify SSH key is not exposed in logs:

Check the workflow run logs for the deploy step. Expected: No unencrypted key material in logs (GitHub Actions automatically redacts secrets, but confirm visually).

---

## Task 10: Set Up SonarQube and GitHub Integration

**Files:**
- Modify: (remote) `/opt/repo_for_lecture/infra/docker-compose.prod.yml` (add SonarQube service if not present)
- Create: `.github/workflows/sonar.yml` (optional; can use existing CI pipeline)

**Interfaces:**
- Consumes: (from Task 7: server infrastructure ready; from Task 6: SafeLine routing ready)
- Produces: SonarQube Community running, GitHub App integrated, PR comments enabled

**Steps:**

- [ ] **Step 1: Add SonarQube to server docker-compose (if not present)**

Check if `infra/docker-compose.prod.yml` includes a SonarQube service. If not, add:

```yaml
sonarqube:
  image: sonarqube:9.9.5-community
  container_name: sonarqube
  environment:
    SONAR_ES_BOOTSTRAP_CHECKS_DISABLED: "true"
    SONAR_JDBC_URL: jdbc:postgresql://postgres:5432/sonarqube
    SONAR_JDBC_USERNAME: postgres
    SONAR_JDBC_PASSWORD: postgres
  ports:
    - "127.0.0.1:9000:9000"
  depends_on:
    - postgres
  restart: unless-stopped
```

Also ensure Postgres has a `sonarqube` database (can be created in a migration or via SQL).

- [ ] **Step 2: Restart server services**

```bash
ssh deploy@72.56.67.176 <<EOF
cd /opt/repo_for_lecture
git pull origin main
docker compose -f infra/docker-compose.prod.yml up -d sonarqube
sleep 30
docker compose -f infra/docker-compose.prod.yml logs sonarqube | tail -20
EOF
```

Expected: SonarQube container is running.

- [ ] **Step 3: Access SonarQube Studio**

Navigate to `https://sonar.notalike.space` in browser. Expected: SonarQube login page. Default credentials: admin / admin.

- [ ] **Step 4: Create SonarQube project and generate token**

In SonarQube Studio:
1. Create a new project: Organization → Projects → Create Project
2. Project key: `repo_for_lecture`
3. Main branch: `main`
4. After project created, go to Administration → Security → Users → Tokens
5. Generate token for GitHub Actions: `github-actions-token`
6. Copy the token

- [ ] **Step 5: Add SONAR_TOKEN to GitHub secrets**

Go to `https://github.com/andreydorozhkin/repo_for_lecture/settings/secrets/actions` and update:

| Secret Name | Value |
|---|---|
| `SONAR_TOKEN` | `<token-from-step-4>` |

- [ ] **Step 6: Install SonarQube GitHub App (if using Community)**

If your SonarQube Community version has branch plugin installed (confirmed in CLAUDE.md), install the GitHub App:
1. SonarQube Studio → Administration → GitHub Settings
2. Authorize GitHub App
3. App will comment on PRs with analysis results

(If Community edition doesn't have this feature natively, the CI workflow will handle posting results via GitHub API instead.)

- [ ] **Step 7: Create SonarQube CI workflow**

Locally, create `.github/workflows/sonar.yml` (or extend existing CI pipeline if one exists):

```yaml
name: SonarQube Analysis

on:
  pull_request:
    branches:
      - main

jobs:
  sonarqube:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Run SonarQube scan
        run: |
          npm install -g sonar-scanner
          sonar-scanner \
            -Dsonar.projectKey=repo_for_lecture \
            -Dsonar.sources=. \
            -Dsonar.host.url=${{ secrets.SONAR_HOST_URL }} \
            -Dsonar.login=${{ secrets.SONAR_TOKEN }}
```

- [ ] **Step 8: Commit and test**

```bash
git add .github/workflows/sonar.yml sonar-project.properties
git commit -m "ci: add SonarQube analysis pipeline"
git push origin main
```

Expected: CI pipeline runs; on next PR, SonarQube posts analysis to the PR.

**Review test for Constraint:** Verify SonarQube port is not exposed externally:

```bash
ssh root@72.56.67.176 "docker inspect sonarqube | grep -A 10 Ports"
```

Expected: Port 9000 bound to `127.0.0.1` only; accessed through SafeLine reverse-proxy.

---

## Task 11: Final Verification and First Demo Branch

**Files:**
- Create: `.github/ISSUE_TEMPLATE/user-story.md` (verify exists; from CLAUDE.md reference)
- Create: (optional) `docs/CONTRIBUTING.md` (outline for student reference)

**Interfaces:**
- Consumes: (from all previous tasks: full infrastructure ready)
- Produces: Confirmation that all systems work end-to-end; template issue created as example for students

**Steps:**

- [ ] **Step 1: Verify all services are accessible through SafeLine**

```bash
curl -s -I https://app.notalike.space | head -1
curl -s -I https://api.notalike.space/rest/v1/ | head -1
curl -s -I https://sonar.notalike.space | head -1
curl -s -I https://safeline.notalike.space | head -1
```

Expected: All return HTTP 200 or 302 (redirect).

- [ ] **Step 2: Verify local stack still works**

```bash
npm run down
npm run up
sleep 20
curl -s http://localhost:3000 | head -5
```

Expected: Local services start; Next.js responds on localhost:3000.

- [ ] **Step 3: Verify user-story template exists**

Check `.github/ISSUE_TEMPLATE/user-story.md` exists (referenced in CLAUDE.md). If not, create it with standard fields:

```markdown
---
name: User Story
about: Create a user story following TDD workflow
title: "[STORY] "
labels: "story"
---

## Role
As a ...

## Goal
I want to ...

## Acceptance Criteria
- [ ] Criterion 1
- [ ] Criterion 2

## Notes
- ...
```

- [ ] **Step 4: Test CD pipeline with a small change**

Create a demo feature branch locally:

```bash
git checkout -b demo/test-cd
echo "# Demo" >> README.md
git add README.md
git commit -m "demo: test CD pipeline"
git push origin demo/test-cd
```

Create a PR on GitHub, then merge to main (or directly push to main if CI passes). Expected: GitHub Actions deploy workflow triggers; app is redeployed on server.

- [ ] **Step 5: Verify merge and deployment**

```bash
# Wait for GitHub Actions to complete
sleep 30
curl -s https://app.notalike.space | grep -q "Demo" && echo "✓ Deployment successful" || echo "✗ Deployment pending"
```

Expected: Change is live on the server.

- [ ] **Step 6: Clean up demo branch**

```bash
git branch -D demo/test-cd
git push origin --delete demo/test-cd
git checkout main
git pull origin main
```

- [ ] **Step 7: Commit final documentation**

```bash
git add .github/ISSUE_TEMPLATE/user-story.md docs/superpowers/plans/2026-11-20-repo-initialization.md
git commit -m "docs: add user story template and initialization plan"
git push origin main
```

**Review test for Constraint:** Confirm all pre-push checks pass:

```bash
npm run lint
npm run typecheck
npm test
npm run test:functions
docker run --rm -v "$PWD":/repo -w /repo ghcr.io/gitleaks/gitleaks:v8.30.1 git --config .gitleaks.toml --redact --no-banner
```

Expected: All pass (green).

---

## Execution Readiness

**All infrastructure and CI/CD is now ready for tomorrow's lecture.**

**For agents (Orca):**
- Create a GitHub Issue using the user-story template
- Mention the story ID in the branch name: `story/<id>-<slug>`
- Write tests from acceptance criteria
- Develop and push commits
- Create PR
- Wait for SonarQube analysis and GitHub Actions CI to complete
- Merge when green

**For manual work:**
- Merge branches with `git merge --no-ff` (preserve history)
- Always ensure CI is green before merging
- Never force-push to main
- Rebase feature branches against main before merging

---
