# Ubuntu Server - 192.168.1.111

## Server Access

**SSH Command:** `ssh 192.168.1.111`
- No username required (uses default SSH key)
- Hostname: `regan-server`
- User: `reganmcgregor`

## System Specifications

- **OS:** Ubuntu 25.04 (Plucky Panda)
- **Kernel:** 6.14.0-37-generic
- **Architecture:** x86_64
- **RAM:** 30 GB (18 GB available)
- **Disk:** 466 GB (83 GB used, 364 GB free)
- **GPU:** NVIDIA GeForce RTX 3060 (12 GB VRAM)

## Running Services

### Supabase (Self-Hosted)
- **Location:** `~/supabase/docker/`
- **Access:** https://supabase.reganmcgregor.com.au
- **Local API:** http://192.168.1.111:8001
- **Kong Port:** 8001 (HTTP), 8444 (HTTPS)
- **Database Port:** 5433 (Supavisor pooler)
- **Components:**
  - PostgreSQL (supabase-db) - Port 5432 (internal)
  - Kong API Gateway (supabase-kong)
  - Studio (supabase-studio)
  - Auth (GoTrue)
  - Storage API
  - Realtime
  - Edge Functions
  - PostgREST
  - Analytics (Logflare)
  - Cloudflare tunnel
- **MCP Access:** Configured in `.mcp.json` at `https://supabase.reganmcgregor.com.au/mcp`
- **Environment:** `~/supabase/.env`
- **Kong Config:** `~/supabase/docker/volumes/api/kong.yml`
- **Database Scripts:** `~/supabase/docker/volumes/db/`

### n8n (Workflow Automation)
- **Location:** `~/n8n/docker-compose.yml`
- **Access:** https://n8n.reganmcgregor.com.au
- **Local Port:** 5678
- **Version:** 2.6.3
- **Database:** PostgreSQL (n8n_postgres_db) - Port 5432
- **Features:**
  - Basic auth enabled
  - Prometheus metrics enabled
  - Trust proxy enabled
  - Cloudflare tunnel configured
- **MCP Access:** Configured in `.mcp.json`

### Qdrant (Vector Database)
- **Container:** `qdrant`
- **Port:** 6333
- **Use Case:** Vector search and embeddings storage
- **Authentication:** API key required (configure `qdrantApi` credential in n8n)
- **Collections:**
  - `product_titles` — 768 dimensions, Cosine distance (auto-created by Seed workflow, used by Product Publisher for RAG title examples)
- **n8n Integration:** Built-in Qdrant Vector Store nodes + Embeddings Ollama sub-nodes

### Open WebUI (AI Interface)
- **Container:** `open-webui`
- **Port:** 8080
- **Image:** CUDA-enabled version (uses RTX 3060)
- **Use Case:** LLM interface

### Ollama (LLM Server)
- **Location:** `~/ollama/docker-compose.yml`
- **Container:** `ollama_cloudflared_tunnel`
- **Config:** `~/.ollama/`
- **Use Case:** Local LLM inference
- **Models:**
  - `llama3.2` — Text generation (Product Publisher title/description)
  - `nomic-embed-text` — 768-dimension embeddings (Qdrant RAG)

### Penpot (Design Tool)
- **Location:** `~/penpot/docker-compose.yaml`
- **Frontend Port:** 9001
- **Components:**
  - Frontend (penpot-penpot-frontend-1)
  - Backend (penpot-penpot-backend-1)
  - PostgreSQL (penpot-penpot-postgres-1)
  - Redis/Valkey (penpot-penpot-valkey-1)
  - Exporter (penpot-penpot-exporter-1)
  - Mailcatcher (port 1080)

### Homepage (Dashboard)
- **Location:** `~/homepage/docker-compose.yml`
- **Port:** 3000
- **Use Case:** Service dashboard/landing page

### Glances (System Monitoring)
- **Location:** `~/glances/docker-compose.yml`
- **Port:** 61208
- **Use Case:** Real-time system monitoring

### Nginx Proxy Manager
- **Location:** `~/nginx-proxy-manager/docker-compose.yml`
- **HTTP Port:** 80
- **HTTPS Port:** 443
- **Admin Port:** 81
- **Use Case:** Reverse proxy and SSL certificate management

### Portainer (Docker Management)
- **Container:** `portainer`
- **HTTPS Port:** 9443
- **HTTP Port:** 8000
- **Use Case:** Docker container management UI

## Docker Networks

- `bridge` - Default bridge network
- `proxy` - Nginx Proxy Manager network
- `n8n_default` - n8n services
- `penpot_penpot` - Penpot services
- `homepage_default` - Homepage service
- `glances_default` - Glances service
- `nginx-proxy-manager_default` - NPM services
- `host` - Host network mode
- `none` - No networking

## Important Directories

- `~/supabase/` - Supabase installation and config
- `~/n8n/` - n8n docker-compose
- `~/ollama/` - Ollama docker-compose
- `~/penpot/` - Penpot design tool
- `~/homepage/` - Homepage dashboard
- `~/glances/` - System monitoring
- `~/nginx-proxy-manager/` - Reverse proxy
- `~/.claude/` - Claude configuration
- `~/.gemini/` - Gemini configuration
- `~/.ollama/` - Ollama models and config

## Domain Routing

All services are routed through Nginx Proxy Manager with Cloudflare tunnels:
- `supabase.reganmcgregor.com.au` → Supabase (Kong port 8001)
- `n8n.reganmcgregor.com.au` → n8n (port 5678)

## Notes

- **Disk:** LVM expanded from 100GB to 466GB (2026-02-04)
- **GPU:** RTX 3060 available for CUDA workloads (Open WebUI, Ollama)
- All services use Docker containers managed through docker-compose
- Cloudflare tunnels provide external access to services
- Most services have dedicated PostgreSQL databases

## Common Commands

```bash
# SSH into server
ssh 192.168.1.111

# View all running containers
docker ps

# Check container logs
docker logs -f <container-name>

# Restart a service (example: Supabase Kong)
cd ~/supabase/docker && docker compose --env-file ~/supabase/.env restart kong

# Execute command in Supabase database
docker exec supabase-db psql -U postgres -d postgres -c "SELECT 1;"

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

*Last updated: 2026-02-04*
*Created by: Claude Code*
