# Archon — Universal AI Agent Context

## What This Project Is

A self-hosted RAG (Retrieval-Augmented Generation) knowledge base tool. Users crawl websites and upload documents; Archon chunks, embeds, and stores them in Supabase. An MCP server exposes the knowledge base to any AI coding assistant (Claude, Cursor, Windsurf, etc.) so agents can search it during coding sessions.

Local-only deployment — each user runs their own instance. Beta software: fix-forward, no backwards compatibility.

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Backend language | Python 3.12 |
| Backend framework | FastAPI |
| Package manager | `uv` |
| Database + vectors | Supabase (PostgreSQL + pgvector) |
| Frontend framework | React + TypeScript (Vite) |
| UI components | Radix UI primitives |
| Data fetching | TanStack Query |
| Styling | Tailwind CSS (Tron-inspired glassmorphism) |
| MCP server | Python (`mcp` library), runs alongside the API |
| Linting (frontend) | Biome (features dir) + ESLint (legacy code) |
| Linting (backend) | Ruff + Mypy |
| Testing (frontend) | Vitest + React Testing Library |
| Testing (backend) | Pytest (async) |
| Container | Docker Compose |

## Architecture

```
Archon/
├── python/
│   └── src/
│       ├── server/
│       │   ├── main.py               # FastAPI app, exception handlers, router includes
│       │   ├── api_routes/           # Route handlers (thin — delegate to services)
│       │   ├── services/             # Business logic layer
│       │   ├── exceptions.py         # Custom exception types
│       │   └── ...
│       └── mcp_server/
│           └── features/[feature]/   # MCP tools grouped by feature
│               └── [feature]_tools.py
└── archon-ui-main/
    └── src/
        ├── features/                 # Vertical slice — each feature owns components/hooks/types/services
        │   ├── [feature]/
        │   │   ├── components/
        │   │   ├── hooks/            # TanStack Query hooks
        │   │   ├── services/         # Frontend API calls
        │   │   └── types/
        │   └── ui/primitives/        # Shared Radix UI wrappers
        └── components/               # Legacy (non-features) components
```

**Request flow**: API Route → Service → Database (Supabase)

**Data fetching rule**: TanStack Query everywhere in `/features`. No prop drilling. No direct fetch calls outside query hooks.

## Database Schema (Supabase)

| Table | Purpose |
|-------|---------|
| `sources` | Crawled sites and uploaded docs — metadata, crawl status, config |
| `documents` | Chunked text with vector embeddings for semantic search |
| `projects` | Optional project management — features array, documents, metadata |
| `tasks` | Task tracking linked to projects — status: todo/doing/review/done; assignee: User/Archon/AI IDE Agent |
| `code_examples` | Extracted code snippets — language, summary, relevance metadata |

## Environment Variables

Required in `python/.env`:

```bash
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_SERVICE_KEY=your-service-key-here
```

See `python/.env.example` for the full list of optional variables.

## Dev Commands

### Recommended: Hybrid mode (backend in Docker, frontend local)

```bash
make dev
# or manually:
docker compose --profile backend up -d
cd archon-ui-main && npm run dev
```

### Backend (python/)

```bash
uv sync --group all                          # Install dependencies
uv run python -m src.server.main             # Run API on :8181
uv run pytest                                # All tests
uv run pytest tests/test_api_essentials.py -v
uv run ruff check                            # Lint
uv run ruff check --fix                      # Auto-fix
uv run mypy src/                             # Type check
```

### Frontend (archon-ui-main/)

```bash
npm run dev                    # Dev server on :3737
npm run build                  # Production build
npm run biome:fix              # Auto-fix /src/features (Biome, 120-char lines)
npm run lint                   # ESLint on legacy code
npx tsc --noEmit               # TypeScript check (all)
npx tsc --noEmit 2>&1 | grep "src/features"  # features only
npm run test                   # Vitest watch mode
```

### Docker (full mode)

```bash
docker compose up --build -d
docker compose logs -f archon-server
docker compose logs -f archon-mcp
docker compose restart archon-server
docker compose down
```

### Linting (both)

```bash
make lint      # Frontend + backend
make lint-fe   # ESLint + Biome
make lint-be   # Ruff + Mypy
```

## Conventions

### Adding an API endpoint
1. Route handler in `python/src/server/api_routes/`
2. Service logic in `python/src/server/services/`
3. Include router in `python/src/server/main.py`
4. Frontend service in `archon-ui-main/src/features/[feature]/services/`

### Adding a UI component (features dir)
1. Use Radix UI primitives from `src/features/ui/primitives/`
2. Component in `src/features/[feature]/components/`
3. Types in `src/features/[feature]/types/`
4. Data via TanStack Query hook in `src/features/[feature]/hooks/`

### Adding or modifying MCP tools
1. Tools live in `python/src/mcp_server/features/[feature]/[feature]_tools.py`
2. Name pattern: `find_[resource]` for list/search/get; `manage_[resource]` for create/update/delete
3. Register in the feature's `__init__.py`

### Code style
- Python: 120-char lines, specific exception types, never catch bare `Exception`, full stack traces with `exc_info=True`
- TypeScript: strict mode, no implicit `any`, double quotes, trailing commas (Biome config)
- Comments document functionality and reasoning only — no references to "beta" or change history
- Never store zero embeddings, null foreign keys, or malformed JSON — skip failed items in batch ops rather than storing corrupted data

### API naming
Use database column values directly in the API and frontend — no mapping layer between DB and FE types.

## Key Files

| File | Purpose |
|------|---------|
| `python/src/server/main.py` | FastAPI app entry point, exception handlers |
| `python/src/server/exceptions.py` | Custom exception hierarchy |
| `python/src/server/api_routes/projects_api.py` | Example route handler |
| `python/src/server/services/project_service.py` | Example service |
| `archon-ui-main/src/features/` | All new UI code lives here |
| `PRPs/ai_docs/ARCHITECTURE.md` | Full architecture reference |
| `PRPs/ai_docs/DATA_FETCHING_ARCHITECTURE.md` | TanStack Query patterns |
| `PRPs/ai_docs/API_NAMING_CONVENTIONS.md` | API naming rules |

## What NOT to Touch

- `archon-ui-main/src/components/` (legacy) — add new code to `/features` instead
- `.env` — never commit; use `python/.env.example` as the template
- `docker-compose.yml` volume config — changing volumes requires `docker compose down -v` and data loss

## Related

- See `CLAUDE.md` for Claude Code-specific configuration
