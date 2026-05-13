# BizComp -- Architecture & Technical Overview

BizComp is an AI-powered intelligence platform for the Belgian corporate registry. It ingests the full KBO/BCE enterprise database (~3 million entities) and the NBB financial filing archive, then exposes this data through a conversational agent that classifies user intent, resolves company references, executes domain-specific tools in parallel, and assembles interactive dashboards -- all streamed to the browser in real time via Server-Sent Events.

This document describes the system end-to-end: what it does, how it works, what problems it solves, and where the tradeoffs are. The source code is private. Nothing here reveals proprietary prompt engineering, extraction schemas, or classification logic.

---

## Table of Contents

1. [System Architecture](#system-architecture)
2. [Data Pipeline](#data-pipeline)
3. [Agent Pipeline](#agent-pipeline)
4. [Frontend](#frontend)
5. [Infrastructure & Deployment](#infrastructure--deployment)
6. [Security](#security)
7. [Testing](#testing)
8. [Tradeoffs & Limitations](#tradeoffs--limitations)

---

## System Architecture

```
                        ┌────────────────────────┐
                        │      User Browser       │
                        └───────────┬────────────┘
                                    │ HTTPS
                        ┌───────────▼────────────┐
                        │   Vercel (Next.js 16)   │
                        │   Frontend + BFF Proxy  │
                        │   Edge Runtime / ISR    │
                        └───────────┬────────────┘
                                    │ x-api-key
                        ┌───────────▼────────────┐
                        │   Caddy Reverse Proxy   │
                        │   Auto HTTPS / HSTS     │
                        └───────────┬────────────┘
                                    │
                        ┌───────────▼────────────┐
                        │   Fastify API Server    │
                        │   Agent Pipeline        │──── Gemini API
                        │   Session Management    │     (classification,
                        │   Financial Engine      │      composition,
                        └──┬──────────┬───────────┘      fallback SQL)
                           │          │
              ┌────────────▼──┐  ┌────▼──────────────┐
              │ PostgreSQL 16 │  │  Python Extractor  │
              │ KBO + Finance │  │  Flask + PyMuPDF   │
              │ Sessions      │  │  Vision API + LLM  │
              └───────────────┘  └────────────────────┘

         Background processes:
         ┌─────────────────────┐  ┌─────────────────────┐
         │  KBO/BCE Importer   │  │  Financial Worker    │
         │  (CLI, SAX parser)  │  │  (polling, refresh)  │
         └─────────────────────┘  └─────────────────────┘
```

### Service Inventory

| Service | Language | Role |
|---|---|---|
| Frontend | TypeScript / Next.js 16 | App shell, SSE streaming, 16 widget types, BFF proxy, SEO pages, admin dashboard |
| Backend | TypeScript / Fastify | Agent pipeline, tool execution, session management, financial engine, REST API |
| PostgreSQL 16 | SQL | KBO enterprise data, financial data cache, sessions, import tracking, geocode cache |
| Python Extractor | Python / Flask | PDF balance sheet extraction via OCR and multimodal LLM |
| KBO Importer | TypeScript (CLI) | Full/daily/catchup imports of KBO XML dumps, reference code imports |
| Financial Worker | TypeScript | Background process polling for stale financial data and re-fetching |
| Caddy | Go | Reverse proxy with automatic HTTPS, security headers |

---

## Data Pipeline

### KBO/BCE Enterprise Data

The Belgian Crossroads Bank for Enterprises (KBO/BCE) publishes the complete enterprise registry as XML dumps, available via SFTP from `ftps.economie.fgov.be`. The dataset contains:

- ~3 million enterprise records (active, stopped, bankrupt, in judicial reorganization)
- Denominations (official names, commercial names, abbreviations) in French, Dutch, and German
- Registered addresses with municipality, postal code, and street-level detail
- NACEBEL activity codes (Belgian extension of NACE Rev. 2)
- Juridical forms, establishment units, contact information
- Mandate holders (directors, administrators, managers) with start/end dates

**Import architecture.** The importer uses SAX-based streaming XML parsers to process the full dump without loading the entire file into memory. This is necessary because the full XML is several gigabytes. The import pipeline supports five modes:

| Mode | Purpose |
|---|---|
| `migrate` | Runs the database schema migration (DDL) |
| `codes` | Imports reference codes (activity codes, juridical forms, status codes) |
| `full` | Full enterprise import from the complete XML dump |
| `daily` | Incremental update from the daily delta XML |
| `catchup` | Replays missed daily updates to close gaps |

The importer also supports automatic SFTP download of the latest ZIP archive. A materialized view (`enterprise_search`) with a GIN-indexed `tsvector` column supports fast full-text company search with ranked results across canonical names, commercial names, and abbreviations.

### NBB Financial Data

Financial data comes from the National Bank of Belgium (NBB) via their CBSO API. Two acquisition strategies run in parallel:

**Bulk XBRL import.** A CLI process downloads batch ZIP archives from the NBB for every business day in the last N years. Each ZIP contains all XBRL filings deposited on that date. The system parses the XBRL rubrics, transforms them into a normalized set of KPIs (revenue, net profit, total assets, equity, FTE headcount, etc.), calculates derived ratios (operating margin, debt-to-equity, current ratio, equity ratio, net profit margin), and upserts the results. The process is resumable via a `financial_year_coverage` table that tracks per-company, per-year completeness.

**On-demand per-company fetch.** When a user queries a company that has no local financial data, the system triggers a real-time fetch. This follows a waterfall:

1. Fetch filing references from the NBB API
2. Attempt XBRL download and parse
3. If XBRL is unavailable or incomplete (common for older filings or micro-entity schemas), fall back to PDF extraction

**PDF extraction cascade.** The Python extractor service implements a 3-layer cascade for extracting structured financial data from Belgian balance sheet PDFs:

| Layer | Method | Speed | Accuracy |
|---|---|---|---|
| 1. Fast path | Text-layer keyword search in the PDF, then OCR only the matched pages via Google Vision API | Fast (~2-5s) | High when text layer exists |
| 2. OCR fallback | Full OCR of all pages, classify each page by type (balance sheet assets, liabilities, income statement, social balance), extract values by coordinate matching | Moderate (~10-30s) | Moderate |
| 3. LLM fallback | Render pages as images, send to Gemini in chunks, parse structured JSON output | Slow (~15-45s) | Good for non-standard layouts |

Each layer validates its output (checks for critical fields like total assets or net profit) and falls through to the next if validation fails. The coordinate-based extraction uses column zone detection to locate the "current year" and "previous year" value columns, then matches accounting codes to their corresponding values by spatial proximity.

**Financial worker.** A background process polls for companies whose financial data is older than 30 days and re-fetches it. This keeps the dataset current without requiring user-triggered refreshes.

---

## Agent Pipeline

Every user query follows a deterministic pipeline. The design philosophy is to avoid LLM calls wherever possible -- the LLM is used for classification and for queries that cannot be planned deterministically, but the majority of the pipeline is pure TypeScript logic.

```
User Message
    │
    ▼
┌──────────────────────────┐
│ Fast Path Check          │ regex detects enterprise number in message
│                          │ → skip LLM entirely, go to deterministic pipeline
└────────────┬─────────────┘
             │ (no match)
             ▼
┌──────────────────────────┐
│ Intent Classification    │ single Gemini call → 17 intents + entity extraction
│                          │ parallel: speculatively resolve active companies
└────────────┬─────────────┘
             │
             ├── greeting → canned response (no LLM)
             ├── clarification → single LLM response
             ├── data_question → conversational answer about on-screen data
             │
             ▼
┌──────────────────────────┐
│ Entity Resolution        │ PostgreSQL full-text search (GIN index)
│                          │ fuzzy matching with confidence scoring
│                          │ disambiguation tables for ambiguous matches
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│ Query Planning           │ deterministic TypeScript mapping:
│                          │ intent + entities → tool calls (no LLM)
└────────────┬─────────────┘
             │
             ├── aggregate/custom query → LLM fallback loop (tool-calling SQL)
             │
             ▼
┌──────────────────────────┐
│ Parallel Tool Execution  │ 13 tools, run concurrently
│                          │ session-level result caching
│                          │ progressive SSE widget_ready events
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│ Dashboard Composition    │ deterministic builders for standard intents
│                          │ localized summary narratives (no LLM)
│                          │ LLM composer only for complex/fallback paths
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│ Suggested Actions        │ context-aware follow-up suggestions
│                          │ filtered by what the user has already asked
│                          │ localized (EN/FR/NL)
└──────────────────────────┘
```

### Stage Details

**Fast path.** When the user message contains an enterprise number (e.g. from clicking a search result), the system skips the LLM classifier entirely. It parses the number with regex, infers the intent from keyword analysis (financial terms, map terms, risk terms, etc.), and routes directly to the deterministic pipeline. This saves ~500ms-1.5s per query.

**Intent classification.** A single Gemini call with a structured system prompt classifies the user's message into one of 17 intent types and extracts entities (company names, person names, NACEBEL sector codes, financial metrics, search filters). The classifier operates at temperature 0.0 for determinism. A post-classification heuristic layer handles Belgian-specific edge cases: Dutch family names that are also company names (Van Oirschot, De Smedt) are disambiguated using commercial suffix detection (BV, NV, SRL, etc.).

**Entity resolution.** Company names extracted by the classifier are resolved to enterprise numbers via PostgreSQL full-text search. The resolver:

- Normalizes input (strips juridical form suffixes, truncates long queries to 4 meaningful words)
- Detects embedded enterprise numbers in mixed strings
- Uses an LRU cache (500 entries, 10-minute TTL) to avoid redundant lookups
- Scores results by relevance and returns disambiguation tables when confidence is low
- Pre-warms the search infrastructure check at startup to avoid cold-start latency

On follow-up queries, entity resolution runs speculatively in parallel with intent classification -- the system resolves the currently active company names while the classifier is still processing, then merges the results.

**Query planning.** A pure TypeScript function maps each intent to a set of tool calls. No LLM involved. The planner handles:

- 14 intent types with specific tool combinations
- Follow-up deduplication (avoids re-fetching the company card if the user already has one on screen)
- Search filter forwarding for multi-turn search refinements
- Fallback detection (aggregate and custom queries that require SQL generation)

**Tool execution.** 13 domain-specific tools execute in parallel:

| Tool | Data Domain | What It Does |
|---|---|---|
| `get_company_card` | Profile | Full company profile: status, legal form, addresses, activities, directors, capital, health score |
| `get_financials` | Financial | 5-year KPI history with ratios, triggers on-demand NBB fetch if data is missing |
| `map_establishments` | Geographic | All establishment units with geocoded addresses (Google Maps API with DB cache) |
| `company_timeline` | Historical | Chronological history: name changes, address changes, activity changes, status changes |
| `company_anomalies` | Intelligence | Health score calculation, anomaly detection (director churn, rapid address changes, negative equity, etc.) |
| `compare_companies` | Comparison | Side-by-side comparison of 2+ companies across all metrics |
| `find_related_companies` | Corporate group | Parent/subsidiary relationships via shared directors and mandates |
| `network_analysis` | Network | Extended graph of corporate connections across multiple hops |
| `investigate_person` | Person | All corporate mandates held by a person across Belgian companies |
| `search_companies` | Search | Filtered company search with 15+ filter dimensions (sector, region, status, age, FTE, revenue, profit, equity, capital, director count, etc.) |
| `sector_financials` | Sector | Aggregate sector benchmarks: median revenue, margin distribution, FTE totals, company rankings |
| `sector_snapshot` | Sector | Quick structural stats: active count, bankruptcies, average age, top locations |
| `sql_query` | Custom | Raw SQL execution against the read-only database (fallback path only) |

Each tool result is streamed to the frontend as a `widget_ready` SSE event as soon as it completes, enabling progressive rendering.

**Dashboard composition.** For standard intents (~90% of queries), the dashboard is assembled entirely in TypeScript -- no LLM call. The deterministic builders:

- Map tool results to widget types (company_card, financial_dashboard, map, timeline, etc.)
- Generate localized summary narratives from the data (e.g. "Revenue at Colruyt grew 12% over 4 years to EUR1.2B, net margins compressed from 3.2% to 2.8%")
- Include contextual information like anomaly counts, health scores, and risk levels
- Handle edge cases: pending financial data, disambiguation tables, partial failures

For aggregate and custom queries, the system falls back to an LLM tool-calling loop where Gemini generates SQL queries, executes them against a read-only database connection, and iterates up to 6 times before synthesizing a final dashboard.

**Session management.** Multi-turn conversations are managed via server-side sessions stored in PostgreSQL:

- Full conversation history (user/agent turn pairs)
- Active entities (company names, enterprise numbers, persons, NACEBEL codes)
- Active search filters (preserved across search refinements)
- Session-level tool cache (avoids re-fetching identical results within a session)
- Last data domains (tracks what is already on screen to avoid redundant widgets)

Follow-up resolution handles pronoun references ("show me their finances") by inheriting the active company from the session context. Search refinements ("now only in Antwerp") merge the new filter with the previous search's full filter set.

---

## Frontend

### Tech Stack

| Layer | Technology |
|---|---|
| Framework | Next.js 16 (App Router) |
| Language | TypeScript 5 |
| UI | React 19, Tailwind CSS 4, Lucide Icons |
| Charts | Recharts 3 |
| Maps | Leaflet + React-Leaflet |
| Graphs | React Flow (@xyflow/react) + Dagre layout |
| Markdown | react-markdown + remark-gfm |
| Internationalization | next-intl 4 (EN/FR/NL) |
| Auth | jose (JWT for admin) |
| Export | html2canvas (PDF/image), CSV export |
| Types | @amarfts/bizcomp-types (shared monorepo package) |

### SSE Streaming & Progressive Rendering

The frontend uses a custom `useAgentStream` hook that manages the full query lifecycle over Server-Sent Events. The stream emits typed events at each pipeline stage:

```
connecting → session → classifying → classified → resolving → resolved
  → executing → widget_ready (×N) → executed → composing → done → result
```

The `widget_ready` event is the key to progressive rendering. As each backend tool completes, its result is streamed to the frontend, which immediately renders the corresponding widget. The user sees the company card appear in ~1 second, followed by the financial dashboard, map, timeline, and other widgets as they resolve -- rather than waiting for the entire pipeline to finish.

The hook also handles:

- Session persistence in `localStorage` with a 24-hour resume window
- Data staleness probing (checks if backend data references are still valid when the tab becomes visible)
- Automatic session retry on stale/expired session errors (transparent to the user)
- Widget history navigation (browse back/forward through previous query results)
- Accumulated entity disambiguation (carries resolved entities across multi-turn interactions)

### Widget Architecture

The Data Canvas renders the agent's dashboard through a `WidgetRenderer` dispatcher that maps each widget type to its React component. 16 widget types:

| Widget | Description |
|---|---|
| `company_card` | Full company profile with status badge, legal form, address, activities, directors, health score, anomaly badges. Supports compact mode for follow-ups. Embeds a sector benchmark strip. |
| `financial_dashboard` | Multi-tab financial analysis: KPIs table, ratio spider chart, trend line charts, year-over-year bar charts. Auto-polls the backend if financial data is still being processed. |
| `compare_dashboard` | Side-by-side company comparison with interactive metric selection and visual diff indicators. |
| `sector_financial_dashboard` | Sector-level overview: aggregate stats, company rankings by metric, revenue distribution charts. |
| `table` | Sortable, filterable, paginated data table with column visibility controls, search, and CSV export. Supports clickable rows for entity navigation. |
| `map` | Interactive Leaflet map with clustered markers for establishment units. |
| `timeline` | Chronological event timeline with categorized entries and date formatting. |
| `corporate_graph` | Interactive force-directed graph of corporate structure (React Flow + Dagre automatic layout). |
| `network_graph` | Extended network visualization showing cross-entity connections and hidden relationships. |
| `insights_card` | Intelligence signals card with severity-colored anomaly list and health score gauge. |
| `sector_benchmark_lite` | Compact sector positioning strip embedded in the company card. |
| `bar_chart` | Recharts bar chart with configurable axes. |
| `pie_chart` | Recharts pie chart. |
| `scorecard` | Single-value KPI card. |
| `markdown` | Rendered markdown narrative (GFM support). |

### SEO & Server Rendering

- Server-rendered company pages (`/company/[id]`) with full metadata, JSON-LD structured data (`WebSite` schema with `SearchAction`), and Open Graph tags
- Dynamic `sitemap.xml` generated from all companies in the database
- Dynamic Open Graph image endpoint for social preview cards
- Canonical URLs and locale alternates for EN/FR/NL
- Deep-linkable analysis pages (`/analysis/[id]`) that auto-trigger the agent pipeline

### Internationalization

Full trilingual support (English, French, Dutch) across:

- All UI strings (managed via `next-intl` JSON message files)
- Streaming stage labels and progress indicators
- Agent summary narratives and contextual hints
- Suggested follow-up actions
- Error messages and disambiguation prompts
- The backend agent responds in the user's detected or preferred language

### Admin Dashboard (Backstage)

Password-protected admin interface with JWT auth:

| Page | What It Shows |
|---|---|
| Overview | LLM queue depth, cache stats, active sessions, shared state store |
| System Health | Auto-refreshing (5s) probes for NBB API, Python extractor, Gemini AI |
| Queries | Full query telemetry log with search, intent filter, date range, pagination |
| Pipeline | Import run history with drill-down into individual run details |
| Analytics | Usage charts (daily queries, intent distribution, language breakdown) |
| Actions | Cache clearing, expired session purge, shared snapshot management |

---

## Infrastructure & Deployment

### Docker Compose Stack

The backend runs as a Docker Compose stack with 6 services:

```yaml
services:
  db:         PostgreSQL 16 (tuned: 1GB shared_buffers, 32MB work_mem, 3GB effective_cache_size)
  backend:    Fastify API server (Node.js)
  importer:   CLI process for KBO data imports (runs on-demand via profiles)
  financial-worker:  Background financial data refresh (polls every 5s)
  python-extractor:  Flask service for PDF extraction (Gemini + Vision API)
  caddy:      Reverse proxy with automatic HTTPS and security headers
```

PostgreSQL is tuned for analytical workloads with large `shared_buffers` and `effective_cache_size` to keep the GIN indexes and materialized views in memory. `random_page_cost` is set to 1.1 to favor index scans on SSD storage.

### Frontend Deployment

The Next.js frontend deploys to Vercel. API routes act as a Backend-for-Frontend (BFF) proxy, forwarding requests to the Fastify backend with the `x-api-key` header injected server-side. This keeps the API key out of the browser.

Vercel-specific optimizations that were implemented to handle traffic within the free tier:

- Edge runtime for all API proxy routes (lower CPU cost than serverless functions)
- Incremental Static Regeneration (ISR) for company pages
- Suppressed internal logging from Next.js data-fetching to reduce CPU overhead

---

## Security

| Layer | Mechanism |
|---|---|
| Authentication | Shared `x-api-key` header for all backend requests; JWT-based admin auth |
| Rate limiting | Configurable per-IP anonymous rate limits |
| Request budget | Per-request timeout (90s default, 180s streaming) with abort control propagated through the entire pipeline |
| Prompt injection | User messages wrapped in `<USER_QUERY>` XML tags; classifier prompt explicitly instructs the LLM to treat the content as data only |
| Database isolation | Agent queries execute against a separate read-only PostgreSQL user with no write permissions |
| Transport | Caddy enforces HTTPS with HSTS, X-Content-Type-Options: nosniff, X-Frame-Options: DENY, Referrer-Policy: no-referrer |
| CORS | Restricted to configured frontend origins |

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

## Tradeoffs & Limitations

This section documents the engineering constraints and compromises in the system. These are not bugs -- they are conscious decisions made under real-world constraints.

### Intent classification is a single LLM call with a long prompt

The classifier prompt is approximately 8KB of structured instructions covering 17 intent types, entity extraction rules, multilingual support, search filter parsing, and security directives. It works well for the Belgian business domain it was designed for, but it is inherently brittle in edge cases:

- Company names that look like person names (common in Belgium where companies are named after founders) require a post-classification heuristic layer that checks for commercial suffixes
- The follow-up vs. data_question vs. new-query distinction relies on the LLM correctly interpreting conversational context, which occasionally fails on ambiguous phrasing
- Adding a new intent type requires careful prompt engineering to avoid regressions in existing intent accuracy

The tradeoff was deliberate: a single-call classifier is ~500ms faster than a multi-stage classification pipeline, and the post-hoc heuristic layer catches the most common misclassifications.

### PDF extraction is a 3-layer cascade because no single approach works

Belgian balance sheets come in dozens of formats: full-schema XBRL, abbreviated XBRL, micro-entity schemas, scanned PDFs, digitally-generated PDFs with inconsistent table layouts, bilingual documents (French/Dutch columns), and occasionally German. No single extraction method handles all of them:

- XBRL parsing is fast and accurate but only available for ~60-70% of filings
- Vision API OCR with coordinate-based extraction works for standard layouts but fails on non-standard column arrangements or when the "current year" / "previous year" headers are missing
- The Gemini multimodal fallback handles non-standard layouts but is slower and more expensive

The cascade approach means that the cheapest, fastest method is tried first, with more expensive fallbacks only triggered when validation fails. The validation gate checks for the presence of critical fields (total assets or net profit) before accepting a result.

### Financial data is hidden until the 5-year window is complete

The system fetches 5 years of financial history per company. Data is not shown to the user until all 5 years have been processed (or confirmed as having no filing). This prevents users from seeing incomplete dashboards where some years are missing mid-sequence. The downside: recently-founded companies that have only 1-2 years of filings may show "data is being retrieved" while the system confirms that the remaining years have no filings.

### The deterministic pipeline eliminates the LLM for ~90% of queries -- but not all

For standard intents (company overview, financials, map, timeline, risk, deep dive, comparison, search, sector benchmark), the entire dashboard is assembled in TypeScript with no LLM call after classification. This saves 1-3 seconds per query and makes the output fully reproducible.

However, aggregate queries ("How many companies went bankrupt in Wallonia last year?") and custom queries that do not map to a predefined tool require the LLM fallback loop, where Gemini generates SQL queries and iterates. This path is slower (~5-15s), less predictable, and can fail if the generated SQL is invalid. A 2-consecutive-error circuit breaker stops the loop and forces a best-effort summary.

### Entity resolution depends on PostgreSQL full-text search

Company name resolution uses a PostgreSQL materialized view with a GIN-indexed `tsvector` column. This is fast (~5-20ms per lookup) but has inherent limitations:

- Very common name fragments ("Transport", "Services", "Group") return many results and trigger disambiguation
- The resolver truncates long search strings to 4 words to avoid expensive multi-token GIN queries, which can lose precision for companies with long names
- Juridical form suffixes (BV, NV, SRL, SA) are stripped before search because they are not part of the canonical name in the database, but this means "Eforge BV" and "Eforge NV" resolve to the same search query

### No horizontal scaling strategy for sessions

Sessions are stored in PostgreSQL. There is no Redis layer, no distributed session store, and no session affinity. This works for the current traffic level but would need to change for multi-instance deployment. The session store does implement expiry and cleanup (via the admin dashboard), but the architecture assumes a single backend instance.

### The frontend is a monolith

All 16 widget types, the chat interface, the admin dashboard, the SEO pages, and the streaming infrastructure live in a single Next.js application. Some of the widget components are large (the financial dashboard is ~37KB, the compare dashboard is ~27KB). Code splitting via Next.js dynamic imports mitigates the bundle size impact, but the codebase would benefit from a more granular component library if it continues to grow.

---

## Scale

Some concrete numbers on the dataset and infrastructure:

| Dimension | Value |
|---|---|
| Enterprise records | ~3 million |
| Denomination records | ~15 million |
| Activity records | ~5 million |
| Financial years tracked | 5 per company (most recent) |
| Widget types | 16 |
| Agent intent types | 17 |
| Agent tools | 13 |
| Search filter dimensions | 15+ |
| Languages supported | 3 (EN, FR, NL) |
| SSE event types | 12 |
| Typical query latency (deterministic path) | 1.5-4s |
| Typical query latency (fallback path) | 5-15s |
| Entity resolution latency | 5-20ms (cached), 20-80ms (uncached) |
| PDF extraction latency (fast path) | 2-5s |
| PDF extraction latency (LLM fallback) | 15-45s |

The system handles real production traffic on the Vercel free tier (frontend) and a single self-hosted server (backend). The Vercel Edge runtime migration was done specifically to reduce CPU exhaustion under traffic spikes.
