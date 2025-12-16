# n8n Sandbox Server Setup

Isolated n8n instance for embedding in node-bench. This setup provides:

- Docker Compose with n8n + Caddy reverse proxy
- Iframe-friendly headers (no X-Frame-Options blocking)
- Multi-layer security (origin whitelist, proxy auth, node blocking)
- Auto-cleanup of old executions

---

## 1. Architecture Overview

```
┌────────────────────────────────────────────────────────────────────┐
│  ALLOWED ORIGINS (whitelist)                                       │
│  ├── https://node-bench.com                                        │
│  ├── https://www.node-bench.com                                    │
│  ├── https://*.vercel.app          (preview deployments)           │
│  └── http://localhost:3000          (local dev)                    │
└────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌────────────────────────────────────────────────────────────────────┐
│  LAYER 1: Nuxt Backend Proxy (/server/api/n8n/[...].ts)            │
│  ───────────────────────────────────────────────────────────────── │
│  • Validates Origin header against whitelist                       │
│  • Validates Referer header                                        │
│  • Adds internal auth token before forwarding                      │
│  • Strips sensitive headers from response                          │
│  • Rate limiting per IP (optional)                                 │
└────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌────────────────────────────────────────────────────────────────────┐
│  LAYER 2: Caddy Reverse Proxy (Hetzner server)                     │
│  ───────────────────────────────────────────────────────────────── │
│  • SSL termination (Let's Encrypt)                                 │
│  • Validates X-NodeBench-Proxy-Token header                        │
│  • Removes X-Frame-Options header                                  │
│  • Sets Content-Security-Policy: frame-ancestors <whitelist>       │
│  • IP whitelist: only accepts from Vercel IPs + your server        │
└────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌────────────────────────────────────────────────────────────────────┐
│  LAYER 3: n8n Container (isolated)                                 │
│  ───────────────────────────────────────────────────────────────── │
│  • Bound to localhost only (not exposed to internet)               │
│  • Dangerous nodes blocked                                         │
│  • Execution timeouts enforced                                     │
│  • Data auto-pruned                                                │
└────────────────────────────────────────────────────────────────────┘
```

---

## 2. Server Requirements (Hetzner)

| Spec         | Recommendation                                       |
| ------------ | ---------------------------------------------------- |
| **Instance** | CX22 (2 vCPU, 4GB RAM) - ~€4/month                   |
| **OS**       | Ubuntu 24.04 LTS                                     |
| **Storage**  | 40GB SSD (default)                                   |
| **Location** | Nuremberg or Falkenstein (EU)                        |
| **Domain**   | Use nip.io (free) or custom domain                   |
| **Firewall** | Allow ports 80, 443 only                             |

### Domain Options

**Option A: nip.io (recommended for testing)**

nip.io is a free wildcard DNS service. Any `<anything>.<IP>.nip.io` resolves to that IP.

```
Server IP: 77.42.27.235
Domain:    77.42.27.235.nip.io
URL:       https://77.42.27.235.nip.io
```

- No DNS configuration needed
- Works with Let's Encrypt SSL
- Perfect for testing before buying a domain

**Option B: Custom domain (production)**

- Point `sandbox.node-bench.com` A record to server IP
- More professional, easier to remember

---

## 3. Security Layers - Detailed

### 3.1 Origin/Referer Validation (Nuxt Proxy)

**Whitelist:**

```
ALLOWED_ORIGINS:
  - https://node-bench.com
  - https://www.node-bench.com
  - https://*.vercel.app          # Preview deployments
  - http://localhost:3000         # Local dev
  - http://localhost:3001         # Alternative local port
```

**Validation logic:**

- Check `Origin` header first
- Fall back to `Referer` header if Origin missing
- Reject with 403 if neither matches whitelist
- Log rejected attempts for monitoring

### 3.2 Proxy Authentication Token

**Purpose:** Ensure only your Nuxt backend can reach the sandbox

```
PROXY_AUTH_TOKEN: <32+ character random string>

Header added by Nuxt proxy:
X-NodeBench-Proxy-Token: <token>

Caddy validates this header before forwarding to n8n
```

### 3.3 IP Whitelisting (Caddy layer)

**Allow only:**

- Vercel's edge IP ranges (for production)
- Your development machine IP (for testing)
- Localhost (for Caddy → n8n internal)

**Vercel IP ranges:** https://vercel.com/docs/security/deployment-protection#ip-addresses

### 3.4 n8n Node Blocking

**Block these dangerous nodes:**

```
NODES_EXCLUDE:
  - n8n-nodes-base.executeCommand     # Shell execution
  - n8n-nodes-base.readWriteFile      # File system access
  - n8n-nodes-base.writeBinaryFile    # Binary file writes
  - n8n-nodes-base.executeWorkflow    # Workflow chaining (escape sandbox)
  - n8n-nodes-base.ssh                # SSH access
  - n8n-nodes-base.ftp                # FTP access
  - n8n-nodes-base.localFileTrigger   # File system watching
```

### 3.5 Execution Limits

```
EXECUTIONS_TIMEOUT: 30              # Max 30 seconds per execution
EXECUTIONS_TIMEOUT_MAX: 60          # Hard limit even if user overrides
N8N_CONCURRENCY_PRODUCTION_LIMIT: 5 # Max 5 concurrent executions
```

### 3.6 Data Pruning (no persistence)

```
EXECUTIONS_DATA_PRUNE: true
EXECUTIONS_DATA_MAX_AGE: 1          # Delete after 1 hour
EXECUTIONS_DATA_PRUNE_MAX_COUNT: 50 # Keep max 50 executions
EXECUTIONS_DATA_SAVE_ON_SUCCESS: none
EXECUTIONS_DATA_SAVE_MANUAL_EXECUTIONS: false
```

---

## 4. HTTP Headers Configuration

### 4.1 Headers to REMOVE (allow iframe)

```
X-Frame-Options: <remove entirely>
```

### 4.2 Headers to ADD

```
Content-Security-Policy: frame-ancestors https://node-bench.com https://www.node-bench.com https://*.vercel.app http://localhost:3000

X-Content-Type-Options: nosniff
Referrer-Policy: strict-origin-when-cross-origin
```

### 4.3 CORS Headers (for API/fetch requests)

```
Access-Control-Allow-Origin: <origin from whitelist>
Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
Access-Control-Allow-Headers: Content-Type, Authorization, X-NodeBench-Token
Access-Control-Allow-Credentials: true
```

---

## 5. n8n Environment Variables - Complete List

```bash
# === CORE ===
N8N_HOST=77.42.27.235.nip.io
N8N_PORT=5678
N8N_PROTOCOL=https
WEBHOOK_URL=https://77.42.27.235.nip.io/
GENERIC_TIMEZONE=Europe/Berlin
TZ=Europe/Berlin

# === SECURITY ===
NODES_EXCLUDE=["n8n-nodes-base.executeCommand","n8n-nodes-base.readWriteFile","n8n-nodes-base.writeBinaryFile","n8n-nodes-base.executeWorkflow","n8n-nodes-base.ssh","n8n-nodes-base.ftp","n8n-nodes-base.localFileTrigger"]

# === EXECUTION LIMITS ===
EXECUTIONS_TIMEOUT=30
EXECUTIONS_TIMEOUT_MAX=60
N8N_CONCURRENCY_PRODUCTION_LIMIT=5

# === DATA PRUNING ===
EXECUTIONS_DATA_PRUNE=true
EXECUTIONS_DATA_MAX_AGE=1
EXECUTIONS_DATA_PRUNE_MAX_COUNT=50
EXECUTIONS_DATA_SAVE_ON_SUCCESS=none
EXECUTIONS_DATA_SAVE_ON_ERROR=all
EXECUTIONS_DATA_SAVE_MANUAL_EXECUTIONS=false

# === DISABLE UNNECESSARY FEATURES ===
N8N_TEMPLATES_ENABLED=false
N8N_VERSION_NOTIFICATIONS_ENABLED=false
N8N_DIAGNOSTICS_ENABLED=false
N8N_PUBLIC_API_DISABLED=true
N8N_PUBLIC_API_SWAGGERUI_DISABLED=true

# === DATABASE ===
DB_TYPE=sqlite  # Simple, no separate DB container needed

# === RUNNERS (code isolation) ===
N8N_RUNNERS_ENABLED=true
```

---

## 6. Nuxt Proxy API Route - Requirements

**Path:** `/server/api/n8n/[...path].ts`

**Responsibilities:**

1. Extract `path` param (everything after `/api/n8n/`)
2. Validate `Origin` or `Referer` against whitelist
3. Add `X-NodeBench-Proxy-Token` header
4. Forward request to `https://sandbox.node-bench.com/{path}`
5. Stream response back (important for SSE/WebSocket)
6. Handle errors gracefully

**Environment variables needed in Nuxt:**

```
N8N_SANDBOX_URL=https://sandbox.node-bench.com
N8N_PROXY_TOKEN=<same token as Caddy expects>
```

---

## 7. Frontend Iframe Component - Requirements

**Props:**

```typescript
interface N8nEmbedProps {
  workflowJson?: object; // Pre-load a workflow
  height?: string; // Default: "600px"
  allowExecution?: boolean; // Default: true
}
```

**Behavior:**

- Renders `<iframe src="/api/n8n/" />`
- Optionally passes workflow via postMessage after load
- Handles n8n events (workflow saved, execution complete) via postMessage

---

## 8. Authentication Strategy

**Option A: Shared guest account (simplest)**

- Create a single `guest@node-bench.com` user in n8n
- Auto-login via URL parameter or cookie set by proxy

**Option B: Disable auth entirely (recommended for sandbox)**

- Set n8n to not require login
- Security relies entirely on proxy layer
- Simpler but requires robust proxy security

**Recommendation:** Option B - security via proxy whitelist

---

## 9. File Structure (to be created)

```
n8n-sandbox/
├── README.md                    # This file
├── .env.example                 # Template environment file
├── docker-compose.yml           # n8n + Caddy services
├── Caddyfile                    # Reverse proxy config
└── scripts/
    └── reset-sandbox.sh         # Cron job to reset data

server/api/n8n/
└── [...path].ts                 # Nuxt proxy route

app/components/
└── N8nEmbed.vue                 # Iframe wrapper component
```

---

## 10. Quick Start

```bash
# 1. Create Hetzner server (CX22, Ubuntu 24.04)
#    Note your server IP (e.g., 5.78.100.123)

# 2. SSH into your server
ssh root@YOUR_SERVER_IP

# 3. Install Docker
curl -fsSL https://get.docker.com | sh

# 4. Clone this repo
git clone git@github.com:lukaskunhardt/n8n-sandbox.git
cd n8n-sandbox

# 5. Configure environment
cp .env.example .env

# 6. Edit .env - set your domain using nip.io
#    Replace YOUR_SERVER_IP with actual IP (e.g., 5.78.100.123)
nano .env
```

**.env example with nip.io:**

```bash
# Use your server IP with nip.io
DOMAIN=YOUR_SERVER_IP.nip.io

# Example: DOMAIN=5.78.100.123.nip.io
```

```bash
# 7. Start services
docker compose up -d

# 8. Check logs
docker compose logs -f

# 9. Test it works
#    Open in browser: https://YOUR_SERVER_IP.nip.io
#    (SSL cert may take 1-2 minutes on first load)
```

---

## 11. Testing Checklist

- [ ] n8n loads in iframe from localhost:3000
- [ ] n8n loads in iframe from production domain
- [ ] n8n loads in iframe from Vercel preview URL
- [ ] Direct access to sandbox.node-bench.com is blocked (no valid proxy token)
- [ ] Blocked nodes don't appear in node panel
- [ ] Execution times out after 30 seconds
- [ ] Old execution data is pruned
- [ ] Workflow can be executed and results viewed
- [ ] WebSocket/SSE connections work (real-time updates)

---

## 12. Cost Estimate

| Item            | Monthly Cost  |
| --------------- | ------------- |
| Hetzner CX22    | ~€4           |
| Domain (if new) | ~€1           |
| **Total**       | **~€5/month** |

# Just clone the submodule repo directly

git clone git@github.com:lukaskunhardt/n8n-sandbox.git
cd n8n-sandbox

# edit .env

docker compose up -d
