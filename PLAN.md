# Droplet Multi-Client Hosting Plan

## Architecture Overview

```
Route53 (NS → Cloudflare) → Cloudflare (proxy/DDOS) → DO Reserved IP → Caddy → Containers
```

## Prerequisites

- Digital Ocean account
- Cloudflare account (free tier is fine)
- Domain registered via AWS (managed in Route53)
- Client site repos on GitHub (private, SSH access)
- Your local machine has the Cloudflare origin cert files ready to copy

---

## Directory Structure on Droplet

```
/srv/
├── caddy/
│   ├── docker-compose.yml
│   ├── Caddyfile
│   └── certs/
│       ├── origin.pem
│       └── origin-key.pem
├── sites/
│   ├── client1/              # cloned from GitHub
│   │   └── docker-compose.yml
│   ├── client2/
│   │   └── docker-compose.yml
│   └── client3/
│       └── docker-compose.yml
└── anythingllm/
    └── instance1/
        └── docker-compose.yml

<YOUR_VOLUME_PATH>/              # DO attached volume (e.g., /mnt/volume_syd1_02)
├── caddy/                       # Caddy TLS state/data
├── anythingllm/
│   └── instance1/               # AnythingLLM database + storage
│   └── instance2/               # (future)
```

Config and code live in `/srv/`. Persistent/generated data lives on the DO volume at `<YOUR_VOLUME_PATH>/`.

---

## Step 1: Create the Droplet and Reserved IP

**Where:** Digital Ocean dashboard (browser)

1. Create a Droplet using the **Docker** image from the DO Marketplace
   - Choose your preferred region and size (recommend at least 4GB RAM to start, given AnythingLLM)
   - Under **Add Volume**, create a volume — DO will automatically format and mount it (typically at `/mnt/<volume-name>`) and add the fstab entry
2. Go to **Networking → Reserved IPs** and create a reserved IP, assign it to your droplet

---

## Step 2: Initial Droplet Setup

**SSH into the droplet from your local machine:**

```bash
ssh root@<YOUR_RESERVED_IP>
```

### 2a. Verify Volume is Mounted

The volume was attached during droplet creation, so it should already be mounted. Run this to see all mounted filesystems:

```bash
df -h
```

Look for your volume (it will be the largest disk that's not the root filesystem). It will be mounted at a path like `/mnt/volume_syd1_02` or `/mnt/data-volume`. Note this path - you'll need it for all subsequent commands in this plan (replace `<YOUR_VOLUME_PATH>` with this path).

You can verify it's correct by running:

```bash
df -h <YOUR_VOLUME_PATH>
```

You should see your volume listed with the correct size.

### 2b. Create Directory Structure

Still on the droplet (as root, in any directory). Replace `<YOUR_VOLUME_PATH>` with your actual volume mount path (e.g., `/mnt/volume_syd1_02`):

```bash
# App directories
mkdir -p /srv/caddy/certs
mkdir -p /srv/sites
mkdir -p /srv/anythingllm/instance1

# Persistent data directories on the volume
mkdir -p <YOUR_VOLUME_PATH>/caddy
mkdir -p <YOUR_VOLUME_PATH>/anythingllm/instance1
```

### 2c. Set Up GitHub SSH Key

Still on the droplet:

```bash
ssh-keygen -t ed25519 -C "droplet-deploy" -f ~/.ssh/id_ed25519 -N ""
cat ~/.ssh/id_ed25519.pub
```

Copy the public key output. In GitHub, go to **Settings → SSH and GPG keys → New SSH key** and paste it. This allows the droplet to clone your private repos.

Test the connection:

```bash
ssh -T git@github.com
```

You should see: "Hi username! You've successfully authenticated..."

### 2d. Create the Docker Network

Still on the droplet:

```bash
docker network create caddy-net
```

---

## Step 3: Cloudflare Setup

**Where:** Cloudflare dashboard (browser) + your local machine terminal

### 3a. Add Domain to Cloudflare

1. Log into Cloudflare → **Add a Site** → enter your `domain.com.au`
2. Choose the **Free** plan
3. Cloudflare will scan existing DNS records — review and confirm
4. Cloudflare gives you two nameservers (e.g., `ada.ns.cloudflare.com`, `bob.ns.cloudflare.com`) — **copy these**

### 3b. Update Nameservers in AWS

**Where:** AWS Console (browser)

1. Go to **Route53 → Registered Domains → your domain**
2. Click **Add or edit name servers**
3. Replace the existing NS records with the two Cloudflare nameservers
4. Save — propagation can take up to 24-48 hours but usually much faster

### 3c. Configure Cloudflare SSL

**Where:** Cloudflare dashboard

1. Go to **SSL/TLS → Overview** → set mode to **Full (Strict)**
2. Go to **SSL/TLS → Origin Server → Create Certificate**
   - Private key type: **RSA (2048)**
   - Hostnames: `*.domain.com.au, domain.com.au` (the wildcard covers all subdomains)
   - Certificate validity: **15 years**
   - Click **Create**
3. You will see two text blocks:
   - **Origin Certificate** — copy and save this as `origin.pem` on your local machine
   - **Private Key** — copy and save this as `origin-key.pem` on your local machine
   - **IMPORTANT:** The private key is only shown once. Save it immediately.

### 3d. Copy Origin Certificates to the Droplet

**Where:** Your local machine terminal

```bash
scp origin.pem root@<YOUR_RESERVED_IP>:/srv/caddy/certs/origin.pem
scp origin-key.pem root@<YOUR_RESERVED_IP>:/srv/caddy/certs/origin-key.pem
```

Then SSH into the droplet and lock down permissions:

```bash
ssh root@<YOUR_RESERVED_IP>
chmod 600 /srv/caddy/certs/origin-key.pem
chmod 644 /srv/caddy/certs/origin.pem
```

### 3e. Create DNS Records in Cloudflare

**Where:** Cloudflare dashboard → DNS → Records

Create the following A records (all with **Proxy status: Proxied** / orange cloud, note the name vlaues are just the sub-domain name, not the full URL):

| Type | Name    | Content (Reserved IP) | Proxy   |
| ---- | ------- | --------------------- | ------- |
| A    | client1 | x.x.x.x               | Proxied |
| A    | client2 | x.x.x.x               | Proxied |
| A    | client3 | x.x.x.x               | Proxied |
| A    | llm1    | x.x.x.x               | Proxied |

Replace `x.x.x.x` with your DO reserved IP.

### 3f. Enable WebSockets

**Where:** Cloudflare dashboard → Network

Ensure **WebSockets** is toggled **ON** (it is by default on most plans).

---

## Step 4: Deploy Caddy

**SSH into the droplet:**

```bash
ssh root@<YOUR_RESERVED_IP>
```

### 4a. Copy Config Files to the Droplet

**Where:** Your local machine terminal

**IMPORTANT:** Before copying, edit `caddy/docker-compose.yml` and replace `<YOUR_VOLUME_PATH>` with your actual volume mount path (e.g., `/mnt/volume_syd1_02`).

Copy the Caddy config files from this repo to the droplet:

```bash
scp caddy/docker-compose.yml root@<YOUR_RESERVED_IP>:/srv/caddy/docker-compose.yml
scp caddy/Caddyfile root@<YOUR_RESERVED_IP>:/srv/caddy/Caddyfile
```

### 4b. Start Caddy

**SSH into the droplet:**

```bash
ssh root@<YOUR_RESERVED_IP>
cd /srv/caddy
docker compose up -d
```

Verify it's running:

```bash
docker ps
docker logs caddy
```

Caddy will show errors about unreachable upstreams (`client1`, `client2`, etc.) — this is expected since those containers don't exist yet.

---

## Step 5: Deploy Client Sites

**SSH into the droplet** (if not already connected):

```bash
ssh root@<YOUR_RESERVED_IP>
```

### 5a. Clone Each Site

**Working directory:** `/srv/sites/`

```bash
cd /srv/sites

git clone git@github.com:your-username/client1-site.git client1
git clone git@github.com:your-username/client2-site.git client2
git clone git@github.com:your-username/client3-site.git client3
```

Each repo should contain at minimum:

- `docker-compose.yml` (see `sites/client1/docker-compose.yml` in this repo for the template)
- Static site files in an `html/` directory

### 5b. Start Each Client

**Working directory:** each client's directory

```bash
cd /srv/sites/client1 && docker compose up -d
cd /srv/sites/client2 && docker compose up -d
cd /srv/sites/client3 && docker compose up -d
```

Verify all containers are running and on the network:

```bash
docker ps
docker network inspect caddy-net
```

---

## Step 6: Deploy AnythingLLM

**SSH into the droplet** (if not already connected):

```bash
ssh root@<YOUR_RESERVED_IP>
```

### 6a. Copy Config Files to the Droplet

**Where:** Your local machine terminal

**IMPORTANT:** Before copying, edit `anythingllm/instance1/docker-compose.yml` and replace `<YOUR_VOLUME_PATH>` with your actual volume mount path (e.g., `/mnt/volume_syd1_02`).

```bash
scp anythingllm/instance1/docker-compose.yml root@<YOUR_RESERVED_IP>:/srv/anythingllm/instance1/docker-compose.yml
```

### 6b. Start AnythingLLM

**SSH into the droplet:**

```bash
cd /srv/anythingllm/instance1
docker compose up -d
```

Verify:

```bash
docker ps
docker logs anythingllm1
```

---

## Step 7: Add the Chat Widget to Client Sites

In each client site's HTML (e.g., `html/index.html` in the client repo), add the embed script before the closing `</body>` tag:

```html
<script
  data-embed-id="unique-embed-id-for-client1"
  data-base-api-url="https://llm1.domain.com.au/api/embed"
  src="https://llm1.domain.com.au/embed/anythingllm-chat-widget.min.js"
></script>
```

- Each client site should have a unique `data-embed-id`
- All three clients sharing instance1 point to `llm1.domain.com.au`
- The embed config (workspace, greeting, etc.) is set up in the AnythingLLM UI at `https://llm1.domain.com.au`

---

## Step 8: Verification

**From the droplet:**

```bash
# Check all containers are running
docker ps

# Check all containers are on caddy-net
docker network inspect caddy-net --format '{{range .Containers}}{{.Name}} {{end}}'

# Check Caddy logs for errors
docker logs caddy
```

**From your local machine / browser:**

1. Visit `https://client1.domain.com.au` — should show the static site
2. Visit `https://client2.domain.com.au` — should show the static site
3. Visit `https://client3.domain.com.au` — should show the static site
4. Visit `https://llm1.domain.com.au` — should show the AnythingLLM setup UI
5. On a client site, check that the chat widget appears and responds
6. Open browser DevTools → Network tab → check for WebSocket connection to `llm1.domain.com.au`

---

## Adding More Clients Later

**SSH into the droplet:**

```bash
ssh root@<YOUR_RESERVED_IP>
```

1. **Clone the new site:**

   ```bash
   cd /srv/sites
   git clone git@github.com:your-username/client4-site.git client4
   cd client4 && docker compose up -d
   ```

2. **If adding a new AnythingLLM instance** (every 3 clients):

   ```bash
   mkdir -p /srv/anythingllm/instance2
   mkdir -p <YOUR_VOLUME_PATH>/anythingllm/instance2
   # Create docker-compose.yml with service name anythingllm2 and update <YOUR_VOLUME_PATH>
   cd /srv/anythingllm/instance2 && docker compose up -d
   ```

3. **Update the Caddyfile** — add the new site blocks to `/srv/caddy/Caddyfile`

4. **Reload Caddy** (no downtime):

   ```bash
   docker exec caddy caddy reload --config /etc/caddy/Caddyfile
   ```

5. **Add DNS records** in Cloudflare for each new subdomain (A record → reserved IP → Proxied)

---

## Updating a Client Site

**SSH into the droplet:**

```bash
ssh root@<YOUR_RESERVED_IP>
cd /srv/sites/client1
git pull
```

Since the HTML is mounted as a volume, nginx serves the updated files immediately — no container restart needed.

---

## Droplet Firewall

**Where:** Digital Ocean dashboard → Networking → Firewalls

Create a firewall and attach to the droplet:

| Direction | Protocol | Port | Sources        |
| --------- | -------- | ---- | -------------- |
| Inbound   | TCP      | 22   | Your IP (SSH)  |
| Inbound   | TCP      | 80   | Cloudflare IPs |
| Inbound   | TCP      | 443  | Cloudflare IPs |
| Outbound  | All      | All  | All            |

Restricting ports 80/443 to Cloudflare IPs ensures traffic can only reach your droplet through Cloudflare's proxy, preventing anyone from bypassing DDOS protection by hitting the reserved IP directly.

Cloudflare publishes their IP ranges at: https://www.cloudflare.com/ips/
