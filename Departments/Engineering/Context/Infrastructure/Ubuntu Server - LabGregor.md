# Ubuntu Server - labgregor

## Server Access

**SSH Command:** `ssh labgregor`
- No username required (uses default SSH key)
- Hostname: `labgregor`
- User: `reganmcgregor`
- LAN IP: `192.168.1.111`
- Tailscale IP: `100.91.31.94` (reachable from anywhere on the tailnet — see below)

## Tailscale (VPN / Remote Access)

- **Version:** 1.98.3
- **Machine name:** `labgregor`
- **Tailscale IPv4:** `100.91.31.94`
- **Tailscale IPv6:** `fd7a:115c:a1e0::8c3a:1f5e`
- **MagicDNS name:** `labgregor.tail86e920.ts.net`
- **Tailnet:** `tail86e920.ts.net`
- **Use Case:** Secure remote access to the server and its services without exposing ports publicly — reach the box over the tailnet from any enrolled device.
- **Other devices on tailnet:**
  - `iphone171` — `100.123.121.50` (iOS)
  - `regans-macbook-pro` — `100.93.253.91` (macOS)
- **Access examples:**
  - `ssh reganmcgregor@100.91.31.94` or `ssh reganmcgregor@labgregor.tail86e920.ts.net`
  - Services are reachable on their host ports over Tailscale, e.g. `http://100.91.31.94:3000` (Homepage), `http://100.91.31.94:8080` (Open WebUI)

### Useful commands

```bash
# Show tailnet status and peers
tailscale status

# Show this machine's Tailscale IPs
tailscale ip -4
tailscale ip -6

# Bring the connection up / down
sudo tailscale up
sudo tailscale down
```

## System Specifications

- **OS:** Ubuntu 25.04 (Plucky Panda)
- **Kernel:** 6.14.0-37-generic
- **Architecture:** x86_64
- **RAM:** 30 GB (~20 GB available)
- **Disk:** 466 GB (140 GB used, 307 GB free — 32% used)
- **GPU:** NVIDIA GeForce RTX 3060 (12 GB VRAM)
- **NVIDIA Driver:** 570.195.03

## Running Services

### Supabase (Self-Hosted)
- **Location:** `~/supabase/docker/`
- **Access:** https://supabase.labgregor.dev
- **Local API:** http://192.168.1.111:8001
- **Kong Port:** 8001 (HTTP), 8444 (HTTPS)
- **Database Port:** 5433 (Supavisor pooler), 6544 (pooler transaction mode)
- **Components:**
  - PostgreSQL (supabase-db) — `supabase/postgres:15.14.1.136`, Port 5432 (internal)
  - Kong API Gateway (supabase-kong) — `kong:3.9.1`
  - Studio (supabase-studio)
  - Auth / GoTrue (supabase-auth) — `gotrue:v2.189.0`
  - Storage API (supabase-storage) — `storage-api:v1.60.4`
  - Realtime (realtime-dev.supabase-realtime) — `realtime:v2.102.3`
  - Edge Functions (supabase-edge-functions) — `edge-runtime:v1.74.0`
  - PostgREST (supabase-rest) — `postgrest:v14.12`
  - Postgres Meta (supabase-meta) — `postgres-meta:v0.96.6`
  - Supavisor Pooler (supabase-pooler) — `supavisor:2.9.5`
  - Analytics / Logflare (supabase-analytics) — Port 4000
  - imgproxy (supabase-imgproxy) — `imgproxy:v3.30.1`
  - Vector log shipper (supabase-vector) — `timberio/vector:0.53.0`
  - Cloudflare tunnel (supabase-cloudflared)
- **MCP Access:** Configured in `.mcp.json` at `https://supabase.labgregor.dev/mcp`
- **Environment:** `~/supabase/.env`
- **Kong Config:** `~/supabase/docker/volumes/api/kong.yml`
- **Database Scripts:** `~/supabase/docker/volumes/db/`

### n8n (Workflow Automation)
- **Location:** `~/n8n/docker-compose.yml`
- **Access:** https://workflows.labgregor.dev
- **Local Port:** 5678
- **Version:** 2.22.5 (`n8nio/n8n:latest`)
- **Database:** PostgreSQL (n8n_postgres_db) — `postgres:16-alpine`, Port 5432
- **Features:**
  - Basic auth enabled
  - Prometheus metrics enabled
  - Trust proxy enabled
  - Cloudflare tunnel configured (n8n_cloudflared_tunnel)
- **MCP Access:** Configured in `.mcp.json`

### Qdrant (Vector Database)
- **Container:** `qdrant` (`qdrant/qdrant:latest`)
- **Port:** 6333 (REST), 6334 (gRPC)
- **Use Case:** Vector search and embeddings storage
- **Authentication:** API key required (configure `qdrantApi` credential in n8n)
- **Collections:**
  - `product_titles` — 768 dimensions, Cosine distance (auto-created by Seed workflow, used by Product Publisher for RAG title examples)
- **n8n Integration:** Built-in Qdrant Vector Store nodes + Embeddings Ollama sub-nodes

### Open WebUI (AI Interface)
- **Container:** `open-webui` (`ghcr.io/open-webui/open-webui:cuda`)
- **Port:** 8080
- **Image:** CUDA-enabled version (uses RTX 3060)
- **Use Case:** LLM interface

### Ollama (LLM Server)
- **Location:** `~/ollama/docker-compose.yml`
- **Container:** `ollama_cloudflared_tunnel`
- **Config:** `~/.ollama/`
- **Use Case:** Local LLM inference (RTX 3060)
- **Models:**
  - `qwen2.5:14b` — 9.0 GB, general text generation
  - `llama3.1:latest` — 4.9 GB, text generation
  - `qwen3:latest` — 5.2 GB
  - `deepseek-r1:latest` — 5.2 GB, reasoning
  - `gemma3:latest` — 3.3 GB
  - `nomic-embed-text:latest` — 274 MB, 768-dimension embeddings (Qdrant RAG)
  - `mxbai-embed-large:latest` — 669 MB, embeddings

### Crawl4AI (Web Scraping / Crawler)
- **Container:** `crawl4ai` (`unclecode/crawl4ai:latest`)
- **Port:** 11235 (API), 6379 (internal Redis)
- **Use Case:** LLM-friendly web crawling and content extraction (feeds n8n / enrichment pipelines)

### Browserless (Headless Chrome)
- **Location:** `~/browserless/docker-compose.yml`
- **Container:** `browserless-browserless-1` (`ghcr.io/browserless/chromium:latest`)
- **Port:** 3100 (maps to internal 3000)
- **Use Case:** Headless browser automation / rendering for scraping workflows

### Penpot (Design Tool)
- **Location:** `~/penpot/docker-compose.yaml`
- **Frontend Port:** 9001 (maps to internal 8080)
- **Components:**
  - Frontend (penpot-penpot-frontend-1)
  - Backend (penpot-penpot-backend-1)
  - PostgreSQL (penpot-penpot-postgres-1) — `postgres:15`
  - Redis/Valkey (penpot-penpot-valkey-1) — `valkey:8.1`
  - Exporter (penpot-penpot-exporter-1)
  - MCP server (penpot-penpot-mcp-1) — `penpotapp/mcp:latest`
  - Mailcatcher (penpot-penpot-mailcatch-1) — port 1080

### Homepage (Dashboard)
- **Location:** `~/homepage/docker-compose.yml`
- **Port:** 3000
- **Use Case:** Service dashboard/landing page

### Glances (System Monitoring)
- **Location:** `~/glances/docker-compose.yml`
- **Port:** 61208
- **Use Case:** Real-time system monitoring (incl. GPU)

### Nginx Proxy Manager
- **Location:** `~/nginx-proxy-manager/docker-compose.yml`
- **Container:** `nginxproxymanager`
- **HTTP Port:** 80
- **HTTPS Port:** 443
- **Admin Port:** 81
- **Use Case:** Reverse proxy and SSL certificate management

### Portainer (Docker Management)
- **Container:** `portainer` (`portainer/portainer-ce:latest`)
- **HTTPS Port:** 9443
- **HTTP Port:** 8000
- **Use Case:** Docker container management UI

## Docker Networks

- `bridge` — Default bridge network
- `proxy` — Shared reverse-proxy network
- `n8n_default` — n8n services
- `penpot_penpot` — Penpot services
- `homepage_default` — Homepage service
- `glances_default` — Glances service
- `nginx-proxy-manager_default` — NPM services
- `browserless_default` — Browserless service
- `host` — Host network mode
- `none` — No networking

## Important Directories

- `~/supabase/` — Supabase installation and config
- `~/n8n/` — n8n docker-compose
- `~/ollama/` — Ollama docker-compose
- `~/browserless/` — Browserless headless Chrome
- `~/penpot/` — Penpot design tool
- `~/homepage/` — Homepage dashboard
- `~/glances/` — System monitoring
- `~/nginx-proxy-manager/` — Reverse proxy
- `~/.claude/` — Claude configuration
- `~/.gemini/` — Gemini configuration
- `~/.ollama/` — Ollama models and config

## Domain Routing

All services are routed through Nginx Proxy Manager with Cloudflare tunnels:
- `supabase.labgregor.dev` → Supabase (Kong port 8001)
- `workflows.labgregor.dev` → n8n (port 5678)

## Notes

- **Disk:** LVM expanded from 100 GB to 466 GB (2026-02-04)
- **GPU:** RTX 3060 available for CUDA workloads (Open WebUI, Ollama)
- All services use Docker containers managed through docker-compose
- Cloudflare tunnels provide external access to services
- Most services have dedicated PostgreSQL databases

## Common Commands

```bash
# SSH into server
ssh labgregor

# View all running containers
docker ps

# Check container logs
docker logs -f <container-name>

# Restart a service (example: Supabase Kong)
cd ~/supabase/docker && docker compose --env-file ~/supabase/.env restart kong

# Execute command in Supabase database
docker exec supabase-db psql -U postgres -d postgres -c "SELECT 1;"

# List Ollama models
docker exec ollama_cloudflared_tunnel ollama list

# Check GPU usage
nvidia-smi

# Check system resources
docker exec glances glances --help
```

## Recent Configurations

### Supabase MCP Setup (2026-01-12)
- Enabled Kong access for MCP endpoint at `/mcp`
- Allowed IPs: 127.0.0.1, ::1, 192.168.1.119 (Mac), 172.18.0.1 (Docker gateway)
- Fixed `supabase_read_only_user` password authentication
- Added user to `~/supabase/docker/volumes/db/roles.sql`
- MCP server has full read/write capabilities via different database users

---

*Last updated: 2026-06-18*
*Created by: Claude Code*
