# Embedding n8n in an Iframe (Without Paying for n8n Embed)

n8n's official Embed product costs $1,260/year minimum. This guide shows how to self-host n8n and embed it in your app for ~€5/month using a Hetzner VPS.

**What we built:** An n8n instance that loads inside an iframe with automatic login—no login screen, users go straight to the workflow editor.

---

## The Problem

When you try to embed self-hosted n8n in an iframe, you hit three blockers:

1. **X-Frame-Options header** - n8n sends `X-Frame-Options: SAMEORIGIN`, browsers refuse to load it
2. **Login screen** - Users see a login form inside the iframe
3. **SameSite cookies** - n8n's auth cookie has `SameSite=Lax`, browsers block it in cross-origin iframes

---

## The Solution

We use Caddy as a reverse proxy to:
- Strip the `X-Frame-Options` header
- Inject a trusted header that auto-logs users in
- Configure n8n to use `SameSite=None` cookies

---

## Step 1: Create a Hetzner Server

1. Go to [Hetzner Cloud Console](https://console.hetzner.cloud/)
2. Create a new project
3. Add a server:
   - **Location:** Nuremberg or Falkenstein
   - **Image:** Ubuntu 24.04
   - **Type:** CX22 (2 vCPU, 4GB RAM) - €4.51/month
   - **SSH Key:** Add your public key
4. Note the server IP (e.g., `77.42.27.235`)

---

## Step 2: SSH into the Server

```bash
ssh root@YOUR_SERVER_IP
```

If you get a host key warning, accept it.

---

## Step 3: Install Docker

```bash
curl -fsSL https://get.docker.com | sh
```

---

## Step 4: Create the Project Directory

```bash
mkdir -p /root/n8n-docker-caddy/caddy_config
mkdir -p /root/n8n-docker-caddy/local_files
cd /root/n8n-docker-caddy
```

---

## Step 5: Create the .env File

We use [nip.io](https://nip.io) for free wildcard DNS. Any `*.YOUR_IP.nip.io` resolves to your IP.

```bash
cat > .env << 'EOF'
DATA_FOLDER=/root/n8n-docker-caddy
DOMAIN_NAME=YOUR_IP.nip.io
SUBDOMAIN=
GENERIC_TIMEZONE=Europe/Berlin
EOF
```

Replace `YOUR_IP` with your actual server IP. For example, if your IP is `77.42.27.235`:

```bash
sed -i 's/YOUR_IP/77.42.27.235/g' .env
```

---

## Step 6: Create docker-compose.yml

```bash
cat > docker-compose.yml << 'EOF'
services:
  caddy:
    image: caddy:latest
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - caddy_data:/data
      - ${DATA_FOLDER}/caddy_config:/config
      - ${DATA_FOLDER}/caddy_config/Caddyfile:/etc/caddy/Caddyfile

  n8n:
    image: docker.n8n.io/n8nio/n8n
    restart: always
    ports:
      - 5678:5678
    environment:
      - N8N_HOST=${SUBDOMAIN}${SUBDOMAIN:+.}${DOMAIN_NAME}
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - NODE_ENV=production
      - WEBHOOK_URL=https://${SUBDOMAIN}${SUBDOMAIN:+.}${DOMAIN_NAME}/
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      - N8N_PROXY_HOPS=1
      # Auto-login via trusted header
      - EXTERNAL_HOOK_FILES=/home/node/.n8n/hooks.js
      - N8N_FORWARD_AUTH_HEADER=X-Sandbox-User
      # CRITICAL: Without this, cookies don't work in cross-origin iframes
      - N8N_SAMESITE_COOKIE=none
    volumes:
      - n8n_data:/home/node/.n8n
      - ${DATA_FOLDER}/local_files:/files
      - ${DATA_FOLDER}/hooks.js:/home/node/.n8n/hooks.js:ro

volumes:
  caddy_data:
    external: true
  n8n_data:
    external: true
EOF
```

---

## Step 7: Create the Caddyfile

Replace `YOUR_IP` with your server IP:

```bash
cat > caddy_config/Caddyfile << 'EOF'
YOUR_IP.nip.io {
    reverse_proxy n8n:5678 {
        flush_interval -1
        # This header triggers auto-login via hooks.js
        header_up X-Sandbox-User contact@node-bench.com
    }

    # Remove the header that blocks iframes
    header -X-Frame-Options

    # Only allow embedding from these origins
    header Content-Security-Policy "frame-ancestors 'self' https://node-bench.com https://www.node-bench.com https://*.vercel.app http://localhost:3000"

    # Standard security headers
    header X-Content-Type-Options "nosniff"
    header Referrer-Policy "strict-origin-when-cross-origin"
}
EOF
```

Replace YOUR_IP:

```bash
sed -i 's/YOUR_IP/77.42.27.235/g' caddy_config/Caddyfile
```

---

## Step 8: Create hooks.js (The Auto-Login Magic)

This hook intercepts requests, looks up the user by the email in the trusted header, and issues an auth cookie.

```bash
cat > hooks.js << 'EOF'
const { dirname, resolve } = require('path')
const Layer = require('router/lib/layer')
const { issueCookie } = require(resolve(dirname(require.resolve('n8n')), 'auth/jwt'))
const ignoreAuthRegexp = /^\/(assets|healthz|webhook|rest\/oauth2-credential)/

module.exports = {
    n8n: {
        ready: [
            async function ({ app }, config) {
                const { stack } = app.router
                const index = stack.findIndex((l) => l.name === 'cookieParser')
                stack.splice(index + 1, 0, new Layer('/', {
                    strict: false,
                    end: false
                }, async (req, res, next) => {
                    if (ignoreAuthRegexp.test(req.url)) return next()
                    if (!config.get('userManagement.isInstanceOwnerSetUp', false)) return next()
                    if (req.cookies?.['n8n-auth']) return next()
                    if (!process.env.N8N_FORWARD_AUTH_HEADER) return next()

                    const email = req.headers[process.env.N8N_FORWARD_AUTH_HEADER.toLowerCase()]
                    if (!email) return next()

                    const user = await this.dbCollections.User.findOneBy({email})
                    if (!user) {
                        res.statusCode = 401
                        res.end('User ' + email + ' not found')
                        return
                    }
                    if (!user.role) user.role = {}

                    issueCookie(res, user)
                    return next()
                }))
            },
        ],
    },
}
EOF
```

---

## Step 9: Create Docker Volumes

```bash
docker volume create caddy_data
docker volume create n8n_data
```

---

## Step 10: Start the Services

```bash
docker compose up -d
```

Check the logs:

```bash
docker compose logs -f
```

Wait until you see:
```
n8n-1  | Editor is now accessible via:
n8n-1  | https://YOUR_IP.nip.io
```

---

## Step 11: Create the Owner Account

**This is critical.** The email you use here MUST match the email in the Caddyfile.

1. Open `https://YOUR_IP.nip.io` in your browser
2. Create an account with email: `contact@node-bench.com` (or whatever you put in the Caddyfile)
3. Complete the setup wizard

---

## Step 12: Test the Auto-Login

```bash
curl -I "https://YOUR_IP.nip.io/"
```

You should see:
- `HTTP/2 200`
- `set-cookie: n8n-auth=...; SameSite=None; Secure`

If you see `SameSite=Lax` instead, the `N8N_SAMESITE_COOKIE=none` env var isn't working. Restart n8n:

```bash
docker compose up -d --force-recreate n8n
```

---

## Step 13: Create the Iframe Test Page

In your frontend app, create a simple test page:

```vue
<script setup lang="ts">
const n8nUrl = "https://YOUR_IP.nip.io";
</script>

<template>
  <div class="min-h-screen bg-gray-900 p-8">
    <h1 class="text-2xl font-bold text-white mb-4">n8n Embed Test</h1>
    <div class="border border-gray-700 rounded-lg overflow-hidden">
      <iframe
        :src="n8nUrl"
        class="w-full"
        style="height: 700px"
        allow="clipboard-read; clipboard-write"
      />
    </div>
  </div>
</template>
```

---

## Troubleshooting

### "User xxx@xxx.com not found"

The email in the Caddyfile doesn't match the owner account email. Either:
- Update the Caddyfile to use the correct email, then `docker compose restart caddy`
- Or delete the n8n volume and recreate the account with the right email

### Login screen still shows

1. Check the cookie has `SameSite=None`:
   ```bash
   curl -I "https://YOUR_IP.nip.io/" | grep -i set-cookie
   ```

2. If it says `SameSite=Lax`, the env var isn't set. Check:
   ```bash
   docker compose exec n8n env | grep SAMESITE
   ```

3. Recreate the container:
   ```bash
   docker compose up -d --force-recreate n8n
   ```

### Iframe blocked

Check the Content-Security-Policy allows your origin:
```bash
curl -I "https://YOUR_IP.nip.io/" | grep -i content-security-policy
```

Add your domain to the `frame-ancestors` list in the Caddyfile.

---

## Critical Gotchas We Discovered

### 1. N8N_SAMESITE_COOKIE=none is essential

Without this, the auth cookie has `SameSite=Lax` and browsers block it in cross-origin iframes. This undocumented env var was found by reading n8n's source code.

### 2. The owner account email must match the Caddyfile

The hooks.js looks up users by email. If the email in `header_up X-Sandbox-User` doesn't match an existing user, you get a 401.

### 3. Don't delete the n8n_data volume

The owner account is stored in SQLite inside this volume. If you delete it, you need to recreate the account.

### 4. flush_interval -1 in Caddy

This enables streaming for Server-Sent Events (SSE), which n8n uses for real-time updates.

---

## File Structure on Server

```
/root/n8n-docker-caddy/
├── .env
├── docker-compose.yml
├── hooks.js
├── caddy_config/
│   └── Caddyfile
└── local_files/
```

---

## Cost

| Item         | Monthly Cost |
|--------------|--------------|
| Hetzner CX22 | €4.51        |
| Domain       | Free (nip.io)|
| **Total**    | **~€5/month**|

Compare to n8n Embed: $105/month ($1,260/year).

---

## Frontend Integration (Nuxt/Node-Bench)

This section is for your main app, not the sandbox server.

### Basic Iframe Embed

Create a page or component that embeds the n8n instance:

```vue
<!-- app/pages/test-embed.vue -->
<script setup lang="ts">
const n8nUrl = "https://YOUR_IP.nip.io";
</script>

<template>
  <div class="min-h-screen bg-gray-900 p-8">
    <h1 class="text-2xl font-bold text-white mb-4">n8n Embed Test</h1>
    <div class="border border-gray-700 rounded-lg overflow-hidden">
      <iframe
        :src="n8nUrl"
        class="w-full"
        style="height: 700px"
        allow="clipboard-read; clipboard-write"
      />
    </div>
  </div>
</template>
```

### Environment Variable (Recommended)

Don't hardcode the URL. Add to your `.env`:

```bash
N8N_SANDBOX_URL=https://YOUR_IP.nip.io
```

Then use it in your component:

```vue
<script setup lang="ts">
const config = useRuntimeConfig();
const n8nUrl = config.public.n8nSandboxUrl;
</script>
```

Add to `nuxt.config.ts`:

```typescript
export default defineNuxtConfig({
  runtimeConfig: {
    public: {
      n8nSandboxUrl: process.env.N8N_SANDBOX_URL || '',
    },
  },
});
```

### Reusable Component

For production, create a reusable component:

```vue
<!-- app/components/N8nEmbed.vue -->
<script setup lang="ts">
defineProps<{
  height?: string;
}>();

const config = useRuntimeConfig();
const n8nUrl = config.public.n8nSandboxUrl;
</script>

<template>
  <div class="border border-gray-700 rounded-lg overflow-hidden">
    <iframe
      v-if="n8nUrl"
      :src="n8nUrl"
      class="w-full"
      :style="{ height: height || '700px' }"
      allow="clipboard-read; clipboard-write"
    />
    <div v-else class="p-4 text-red-500">
      N8N_SANDBOX_URL not configured
    </div>
  </div>
</template>
```

Use it anywhere:

```vue
<N8nEmbed height="800px" />
```

### Loading a Specific Workflow

To open a specific workflow, append the workflow ID to the URL:

```vue
<iframe :src="`${n8nUrl}/workflow/abc123`" />
```

### Communicating with the Iframe (Future)

n8n doesn't have a postMessage API, but you could potentially:
- Pre-load workflows by navigating to specific URLs
- Use URL parameters if n8n supports them
- Build a custom n8n node that communicates with your app

---

## Security Considerations

This setup trusts anyone who can reach the server. For production:

1. Add IP whitelisting in Caddy to only allow requests from your app's servers
2. Use a proxy token that your backend adds to requests
3. Block dangerous n8n nodes (executeCommand, ssh, ftp, etc.)

See the original architecture diagram at the top of the old README for the full production security model.
