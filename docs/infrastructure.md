# Infrastructure

This page covers deployment, database tuning, security, and testing.

---

## Docker Compose Stack

The backend runs as a Docker Compose stack with 6 services:

| Service | Image / Build | Role |
|---|---|---|
| `db` | `postgres:16` | Primary database for all enterprise, financial, and session data |
| `backend` | Custom (Node.js) | Fastify API server running the agent pipeline |
| `importer` | Custom (Node.js) | CLI process for KBO data imports (runs on-demand via Docker profiles) |
| `financial-worker` | Custom (Node.js) | Background financial data refresh (polls every 5s) |
| `python-extractor` | Custom (Python) | Flask service for PDF extraction (Gemini + Vision API) |
| `caddy` | `caddy:2-alpine` | Reverse proxy with automatic HTTPS and security headers |

### PostgreSQL Tuning

PostgreSQL is tuned for analytical workloads:

| Parameter | Value | Rationale |
|---|---|---|
| `shared_buffers` | 1GB | Keep GIN indexes and materialized views in memory |
| `work_mem` | 32MB | Support complex sort/join operations in agent queries |
| `effective_cache_size` | 3GB | Inform the query planner about available OS cache |
| `maintenance_work_mem` | 256MB | Speed up materialized view refreshes and index rebuilds |
| `random_page_cost` | 1.1 | Favor index scans on SSD storage |
| `shm_size` | 1GB | Adequate shared memory for large result sets |

### Service Dependencies

```
db (healthy) ──► backend ──► caddy
             ──► importer
             ──► financial-worker ──► python-extractor
```

The importer runs on-demand via Docker Compose profiles (`docker compose --profile cli run importer`). All other services run continuously with `restart: unless-stopped`.

---

## Frontend Deployment

The Next.js frontend deploys to Vercel. API routes act as a Backend-for-Frontend (BFF) proxy, forwarding requests to the Fastify backend with the `x-api-key` header injected server-side. This keeps the API key out of the browser.

### Vercel Optimizations

These were implemented specifically to handle production traffic within the free tier:

| Optimization | Impact |
|---|---|
| Edge runtime for all API proxy routes | Lower CPU cost than serverless functions |
| Incremental Static Regeneration (ISR) for company pages | Avoids re-rendering on every request |
| Suppressed internal logging from Next.js data-fetching | Reduces CPU overhead from ISR background revalidation |

---

## Security

| Layer | Mechanism |
|---|---|
| **Authentication** | Shared `x-api-key` header for all backend requests; JWT-based admin auth |
| **Rate limiting** | Configurable per-IP anonymous rate limits |
| **Request budget** | Per-request timeout (90s default, 180s streaming) with abort control propagated through the entire pipeline |
| **Prompt injection** | User messages wrapped in `<USER_QUERY>` XML tags; classifier prompt instructs the LLM to treat the content as data only |
| **Database isolation** | Agent queries execute against a separate read-only PostgreSQL user with no write permissions |
| **Transport** | Caddy enforces HTTPS with HSTS, X-Content-Type-Options: nosniff, X-Frame-Options: DENY, Referrer-Policy: no-referrer |
| **CORS** | Restricted to configured frontend origins |
| **Secrets** | API keys, database credentials, and GCP service account keys are injected via environment variables and bind-mounted secrets, never committed to source control |

---

## Testing

| Suite | What It Covers | Runner |
|---|---|---|
| Query planner | Intent-to-tool mapping for all 14+ intent types | Vitest |
| Suggested actions | Context-aware follow-up generation across intents and entity types | Vitest |
| Session context | Follow-up inheritance, entity accumulation, search filter merging | Vitest |
| Tool states | State machine transitions (ok, error, ambiguous, pending) | Vitest |
| Follow-up normalization | Pronoun resolution, intent flattening, entity carry-over | Vitest |
| Rate limiting | Per-IP request limits, anonymous access policies | Vitest |
| Request budget | Timeout enforcement, abort signal propagation | Vitest |
| Agent regression | End-to-end pipeline tests across representative query types | Custom harness |
| Agent E2E | Full pipeline with live database and API keys | Manual |

---

**Previous:** [Frontend](frontend.md) | **Next:** [Tradeoffs](tradeoffs.md)
