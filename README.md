# Droplet - Multi-Client Hosting Setup

Configuration and deployment files for hosting multiple client sites on a single Digital Ocean droplet, each with a scraped static website and a connected [AnythingLLM](https://anythingllm.com/) chat widget.

## What This Does

Each client gets:

- A **static website** (scraped web pages) served via nginx, accessible at `clientname.yourdomain.com.au`
- An **AnythingLLM chat widget** embedded on the site, powered by a shared AnythingLLM instance that provides RAG-based Q&A about the client's content

Traffic flows through:

```
Route53 (NS → Cloudflare) → Cloudflare (proxy/DDOS) → DO Reserved IP → Caddy → Containers
```

## Architecture

All services run as Docker containers on a single droplet, connected via a shared `caddy-net` network:

- **Caddy** - Reverse proxy handling TLS termination with Cloudflare origin certificates and routing subdomains to the correct containers
- **Client sites** - nginx containers serving static HTML (one per client)
- **AnythingLLM** - RAG/chat instances (one instance per ~3 clients)

## Directory Structure

```
/srv/                              # Config and code (on droplet)
├── caddy/
│   ├── docker-compose.yml
│   ├── Caddyfile
│   └── certs/
│       ├── origin.pem
│       └── origin-key.pem
├── sites/
│   ├── client1/                   # Cloned from GitHub
│   │   ├── docker-compose.yml
│   │   └── html/
│   ├── client2/
│   └── client3/
└── anythingllm/
    └── instance1/
        └── docker-compose.yml

<VOLUME_PATH>/                     # Persistent data (DO attached volume)
├── caddy/                         # Caddy TLS state
└── anythingllm/
    └── instance1/                 # AnythingLLM database + uploads
```

## Repository Contents

| Path                                       | Purpose                                                |
| ------------------------------------------ | ------------------------------------------------------ |
| `caddy/docker-compose.yml`                 | Caddy reverse proxy container config                   |
| `caddy/Caddyfile`                          | Routing rules (subdomain → container)                  |
| `sites/client*/docker-compose.yml`         | Template docker-compose for nginx client sites         |
| `anythingllm/instance1/docker-compose.yml` | AnythingLLM container config                           |
| `tutorial.html`                            | Interactive step-by-step setup guide (open in browser) |
| `PLAN.md`                                  | Detailed deployment plan with all commands             |

## Getting Started

Open `tutorial.html` in your browser for an interactive 18-step walkthrough that covers:

1. Creating the droplet and reserved IP
2. Setting up the directory structure and SSH keys
3. Configuring Cloudflare DNS and SSL certificates
4. Deploying Caddy, client sites, and AnythingLLM
5. Adding the chat widget to client sites
6. Verification and firewall setup

The tutorial lets you enter your specific values (IP, domain, subdomains, volume path) and interpolates them into all commands, so you can copy-paste directly.

## Prerequisites

- Digital Ocean account
- Cloudflare account (free tier)
- Domain registered via AWS Route53
- Client site repos on GitHub (private, SSH access)

## Adding More Clients

1. Use the [Website Snapshot](https://github.com/bennythemink/website-snapshot/tree/main) project to download a copy of the site in question.
2. This will download and save a copy of the site as well as creating the dockerfile and docker-compose files.
3. SSH into the server and clone the new site repo to `/srv/sites/` on the droplet.
4. Start the container with `docker compose up -d`
5. Navigate tot he Caddyfile and add a new proxy block pointing to the new container/service name.
6. Reload Caddy: `docker exec caddy caddy reload --config /etc/caddy/Caddyfile`
7. Add a DNS record in Cloudflare (A record, proxied)

## Updating a Client Site

```bash
ssh root@<YOUR_RESERVED_IP>
cd /srv/sites/client1
git pull
```

Since HTML is mounted as a volume, nginx serves the updated files immediately with no container restart needed.
