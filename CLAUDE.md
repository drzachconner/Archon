# Archon — AI Coding OS

## Project Overview

Archon is Cole Medin's "Operating System for AI Coding" — a RAG knowledge base, context manager, and task tracker that connects to Claude Code (and other AI coding assistants) via an MCP server. It lets you crawl docs, upload files, and manage tasks, all queryable by your AI agents at coding time. Deployed locally via Docker; each user runs their own instance.

**Status**: Cloned and forked. NOT yet running. Activation requires a Supabase project and Docker Desktop. See the Activation Steps section below.

---

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| **Frontend** | React + Vite + TypeScript + TailwindCSS (port 3737) |
| **API Server** | FastAPI + Socket.IO + Python 3.12 (port 8181) |
| **MCP Server** | Lightweight FastAPI HTTP wrapper (port 8051) |
| **Agents Service** | PydanticAI agents for RAG/reranking (port 8052) |
| **Agent Work Orders** *(optional)* | Claude Code CLI automation (port 8053) |
| **Database** | Supabase (PostgreSQL + PGVector for embeddings) |
| **Embeddings** | OpenAI (default), Gemini, or Ollama |
| **Package mgr (BE)** | `uv` |
| **Package mgr (FE)** | npm |
| **Containerization** | Docker Compose |

---

## Architecture

### Microservices

```
┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│  Frontend (UI)   │    │  Server (API)    │    │   MCP Server     │    │ Agents Service   │
│  React + Vite    │◄──►│  FastAPI +       │◄──►│  MCP tools +     │◄──►│  PydanticAI,     │
│  Port 3737       │    │  SocketIO        │    │  SSE/stdio       │    │  reranking       │
│                  │    │  Port 8181       │    │  Port 8051       │    │  Port 8052       │
└──────────────────┘    └──────────────────┘    └──────────────────┘    └──────────────────┘
         │                       │                       │                        │
         └───────────────────────┴───────────────────────┴────────────────────────┘
                                           │
                                  ┌─────────────────┐
                                  │    Supabase      │
                                  │  PostgreSQL +    │
                                  │  PGVector        │
                                  └─────────────────┘
```

### Service Responsibilities

| Service | Directory | Purpose |
|---------|-----------|---------|
| Frontend | `archon-ui-main/` | Dashboard, knowledge base UI, project management |
| Server | `python/src/server/` | Core business logic, web crawling, doc processing |
| MCP Server | `python/src/mcp/` | MCP protocol interface for AI clients |
| Agents | `python/src/agents/` | Document + RAG agents, streaming responses |
| Agent Work Orders | `python/src/agent_work_orders/` | Claude Code CLI workflow execution *(optional)* |

### Key Database Tables

- `sources` — crawled websites and uploaded documents
- `documents` — chunked text with vector embeddings
- `projects` — project management (optional feature)
- `tasks` — todo/doing/review/done task tracker
- `code_examples` — extracted code snippets for RAG

---

## Directory Structure

```
Archon/
├── archon-ui-main/        # React + Vite frontend
│   └── src/features/      # Vertical slice architecture
├── python/
│   └── src/
│       ├── server/        # Core API + business logic
│       ├── mcp/           # MCP server
│       ├── agents/        # PydanticAI agents
│       └── agent_work_orders/  # Optional workflow engine
├── migration/
│   ├── complete_setup.sql # Run once in Supabase to create all tables
│   └── RESET_DB.sql       # Wipe Archon tables (destructive)
├── PRPs/ai_docs/          # Architecture reference docs (ARCHITECTURE.md, etc.)
├── docker-compose.yml     # Main service orchestration
├── Makefile               # Developer shortcuts
├── .env.example           # Template for all environment variables
└── CLAUDE.md              # This file
```

---

## Activation Steps (Dormant → Running)

Archon is currently **not running**. Follow these steps to activate:

### Prerequisites
- Docker Desktop installed and running
- Supabase account (free tier works)
- OpenAI API key (or Gemini/Ollama — configured later via UI)

### Step 1: Create Supabase Project
1. Go to [supabase.com/dashboard](https://supabase.com/dashboard) → New Project → name it "Archon"
2. Wait for project to initialize (~2 min)
3. Go to **Settings → API keys** and copy:
   - **Project URL** (SUPABASE_URL)
   - **service_role key** — the LONGER one (NOT the anon key — anon will break all saves)

### Step 2: Run Database Migration
1. In Supabase dashboard → **SQL Editor**
2. Copy and paste the full contents of `migration/complete_setup.sql`
3. Click Run — this creates all tables, functions, and policies

### Step 3: Configure Environment
```bash
cd ~/Code/Archon
cp .env.example .env
# Edit .env and fill in:
# SUPABASE_URL=https://your-project.supabase.co
# SUPABASE_SERVICE_KEY=eyJ... (the long service_role key)
```
Store the Supabase credentials in 1Password under "Archon" before adding to `.env`.
The `.env` file is gitignored — never commit it.

### Step 4: Start Services
```bash
docker compose up --build -d
```
This builds and starts all core containers: archon-ui (3737), archon-server (8181), archon-mcp (8051), archon-agents (8052).

### Step 5: Complete Onboarding
1. Open [http://localhost:3737](http://localhost:3737)
2. Archon's onboarding flow will prompt for an LLM API key (OpenAI default)
3. Store that API key in 1Password under "Archon → OpenAI Key" before entering it in the UI

### Step 6: Connect to Claude Code
1. In Archon UI → **MCP Dashboard** → copy the MCP connection config
2. Add the config to `~/.claude/settings.json` under `mcpServers`
3. Typical config:
   ```json
   "archon": {
     "type": "sse",
     "url": "http://localhost:8051/sse"
   }
   ```
4. Restart Claude Code — Archon MCP tools will be available

### Verification
```bash
curl http://localhost:8181/health   # API server
curl http://localhost:8051/health   # MCP server
```

---

## Development Conventions

- **CommonJS → no**: Python backend only. Frontend is ESM/TypeScript.
- **Backend**: Python 3.12, `uv` package manager, 120-char line length
- **Frontend**: TypeScript strict mode, Biome for `/src/features/`, ESLint for legacy `/src/components/`
- **Architecture pattern**: Vertical slices in `/features` — each feature owns components, hooks, services, types
- **Data fetching**: TanStack Query everywhere — no prop drilling, no direct API calls in components
- **Service layer**: API Route → Service → Database (never skip the service layer)
- **Error handling**: Fail fast and loud for startup/config/auth failures; continue with logging for batch ops

---

## Environment Variables

Required in `.env` (see `.env.example` for full reference):

| Variable | Required | Purpose |
|----------|----------|---------|
| `SUPABASE_URL` | Yes | Supabase project URL |
| `SUPABASE_SERVICE_KEY` | Yes | Service role key (long key, not anon) |
| `ANTHROPIC_API_KEY` | Optional | For Agent Work Orders feature |
| `CLAUDE_CODE_OAUTH_TOKEN` | Optional | Claude Code OAuth (alternative to API key) |
| `GITHUB_PAT_TOKEN` | Optional | For Agent Work Orders PR creation |
| `HOST` | No | Hostname (default: localhost) |
| `ARCHON_UI_PORT` | No | UI port (default: 3737) |
| `ARCHON_SERVER_PORT` | No | API port (default: 8181) |
| `ARCHON_MCP_PORT` | No | MCP port (default: 8051) |
| `ARCHON_AGENTS_PORT` | No | Agents port (default: 8052) |
| `ENABLE_AGENT_WORK_ORDERS` | No | Enable optional workflow engine (default: false) |
| `LOG_LEVEL` | No | Logging level (default: INFO) |

All LLM API keys (OpenAI, Gemini, OpenRouter) and RAG settings are managed via the Archon Settings UI after startup — not in `.env`.

---

## Workflow (Common Operations)

```bash
# Start all services
docker compose up -d

# Hybrid dev mode (backend Docker, frontend local with hot reload)
make dev

# Stop all services
docker compose down

# View logs
docker compose logs -f archon-server
docker compose logs -f archon-mcp

# Rebuild after code changes
docker compose up --build -d

# Upgrade Archon (pull latest from upstream)
git fetch upstream && git merge upstream/main
docker compose up --build -d
# Then check Settings → Database Migrations for any pending SQL

# Backend linting
cd python && uv run ruff check
cd python && uv run mypy src/

# Frontend linting
cd archon-ui-main && npm run biome:fix
```

---

## Known Issues

1. **Not yet activated** — Supabase project not created, `.env` not populated, Docker not started. See Activation Steps.
2. **Beta software** — Archon is in beta. Expect rough edges. Cole's team follows fix-forward, no backward compat.
3. **MCP context tax** — When connected, all Archon MCP tool schemas inject ~100-120 tokens per turn. Only connect when actively using the knowledge base. Disconnect from `settings.json` when not in use.
4. **Service role key confusion** — Using the Supabase anon key instead of service_role key causes all saves to silently fail. Use the longer key labeled "service_role" in Supabase dashboard → API keys.

---

## Security

- `.env` is gitignored — never commit it
- Store Supabase URL + service_role key in 1Password under "Archon" item
- Store LLM API keys in 1Password (entered in Archon UI, stored encrypted in Supabase)
- `ENABLE_DOCKER_SOCKET_MONITORING` defaults to `false` — keep it that way (Docker socket = root-equivalent host access)
- MCP server runs on localhost only (port 8051) — not exposed to internet

---

## Subagent Orchestration

| Subagent | When to Use |
|----------|-------------|
| **codebase-explorer** | Before modifying any service — understand how FastAPI routes, services, and TanStack Query hooks interact |
| **browser-navigator** | To test the Archon UI locally once running |
| **pre-push-validator** | Before pushing changes — verify no secrets leaked and types pass |
| **security-scanner** | Before enabling Agent Work Orders feature — it executes Claude Code CLI commands |

---

## GSD + Teams Strategy

Not applicable for normal Archon usage — it's a third-party tool we're running, not building. For any customization or bug fixes, work directly in the main agent context; changes are typically isolated to one service.

If contributing significant features upstream, use GSD phases: plan → implement one service → test → PR.

---

## MCP Connections

This project IS an MCP server. When activated, Claude Code connects to it:
- **Connection type**: SSE (Server-Sent Events)
- **MCP URL**: `http://localhost:8051/sse`
- **Add to**: `~/.claude/settings.json` → `mcpServers`

---

## Remotes

- `origin` → `https://github.com/drzachconner/Archon.git` (personal fork)
- `upstream` → `https://github.com/coleam00/Archon.git` (Cole's repo — pull updates from here)

---

## Completed Work

- Cloned from `coleam00/Archon` (main branch)
- CLAUDE.md created with all 14 standard sections
- Forked to `drzachconner/Archon`
- Upstream remote configured for receiving Cole's updates
