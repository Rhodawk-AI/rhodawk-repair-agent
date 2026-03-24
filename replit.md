# Workspace

## Overview

pnpm workspace monorepo using TypeScript. Each package manages its own dependencies.
The main application is **TestFixLoop** — an autonomous CI/CD code stabilization agent.

## Stack

- **Monorepo tool**: pnpm workspaces
- **Node.js version**: 24
- **Package manager**: pnpm
- **TypeScript version**: 5.9
- **API framework**: FastAPI (Python 3.11) — the primary API backend
- **Database**: PostgreSQL + Drizzle ORM (unused in TestFixLoop, available for extension)
- **Validation**: Zod (`zod/v4`), `drizzle-zod`
- **API codegen**: Orval (from OpenAPI spec)
- **Build**: esbuild (CJS bundle) for Express; uvicorn for Python

## Structure

```text
artifacts-monorepo/
├── artifacts/              # Deployable applications
│   ├── api-server/         # REPLACED: now points to Python FastAPI backend
│   ├── testfixloop/        # React + Vite frontend (TestFixLoop dashboard)
│   └── testfixloop-backend/ # Python FastAPI backend (main.py)
├── lib/                    # Shared libraries
│   ├── api-spec/           # OpenAPI spec + Orval codegen config
│   ├── api-client-react/   # Generated React Query hooks
│   ├── api-zod/            # Generated Zod schemas from OpenAPI
│   └── db/                 # Drizzle ORM schema + DB connection
├── scripts/                # Utility scripts
├── pnpm-workspace.yaml     # pnpm workspace
├── tsconfig.base.json      # Shared TS options
├── tsconfig.json           # Root TS project references
└── package.json            # Root package with hoisted devDeps
```

## TestFixLoop Application

### What It Does
Autonomous CI/CD agent that:
1. Clones a GitHub repo
2. Installs dependencies (7-strategy fallback installer)
3. Runs pytest and captures all failures
4. Uses a 7-tier LLM provider ladder (Gemini → DeepSeek → Groq → Qwen/OpenRouter → GitHub Models → OpenRouter Free → Claude) to fix broken files
5. Commits fixes back to GitHub
6. Re-clones and re-runs tests
7. Repeats until 0 test failures or max cycles reached
8. Produces a downloadable fixed codebase ZIP and stabilization report

### Architecture

**Frontend** (`artifacts/testfixloop/`):
- React + Vite + Tailwind (dark hacker terminal theme, neon green)
- React Query for job polling (every 3s while running)
- SSE EventSource for live log streaming
- Recharts for per-cycle test pass/fail bar chart and token usage area chart
- Framer Motion for animations

**Backend** (`artifacts/testfixloop-backend/main.py`):
- Python 3.11 + FastAPI
- LiteLLM unified LLM routing layer
- GitPython for clone/commit/push
- asyncio background tasks per job
- Jobs stored in-memory (in /tmp/testfixloop_jobs)
- SSE streaming of log entries as JSON

### Provider Ladder (in priority order)
1. Gemini Pro (>500KB payloads)
2. Gemini Flash (50–500KB)
3. DeepSeek V3 (deepseek-chat)
4. DeepSeek R1 (deepseek-reasoner, for stuck cycles)
5. Groq LLaMA (default for <50KB payloads)
6. Qwen3 Coder 480B via OpenRouter (free)
7. Qwen3 235B via OpenRouter (free)
8. GitHub Models GPT-4o (uses GitHub PAT)
9. OpenRouter free rotation (llama-3.1-8b)
10. Claude (optional, only if ANTHROPIC_API_KEY present)

### Workflows
- `artifacts/api-server: API Server` → runs `python main.py` (port 8080, serves `/api`)
- `artifacts/testfixloop: web` → runs Vite dev server (port 25726, serves `/`)

### Key API Endpoints
- `POST /api/jobs` — create a new stabilization job
- `GET /api/jobs/{jobId}` — get job status, cycle data, provider status
- `GET /api/jobs/{jobId}/logs` — SSE stream of log entries
- `GET /api/jobs/{jobId}/download/{zip|report}` — download output files
- `POST /api/providers/discover` — discover free OpenRouter models

## TypeScript & Composite Projects

Every package extends `tsconfig.base.json` which sets `composite: true`. The root `tsconfig.json` lists all packages as project references.

- **Always typecheck from the root** — run `pnpm run typecheck`
- **`emitDeclarationOnly`** — we only emit `.d.ts` files during typecheck

## Root Scripts

- `pnpm run build` — runs `typecheck` first, then recursively runs `build` in all packages
- `pnpm run typecheck` — runs `tsc --build --emitDeclarationOnly` using project references
