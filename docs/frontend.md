# Frontend

The frontend is a Next.js 16 application that serves as the primary user interface, a Backend-for-Frontend proxy, and the SEO rendering layer.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | Next.js 16 (App Router) |
| Language | TypeScript 5 |
| UI | React 19, Tailwind CSS 4, Lucide Icons |
| Charts | Recharts 3 |
| Maps | Leaflet + React-Leaflet |
| Network Graphs | React Flow (@xyflow/react) + Dagre automatic layout |
| Markdown | react-markdown + remark-gfm |
| Internationalization | next-intl 4 (EN/FR/NL) |
| Auth | jose (JWT for admin) |
| Export | html2canvas (PDF/image), CSV export |
| Types | @amarfts/bizcomp-types (shared monorepo package) |

---

## SSE Streaming and Progressive Rendering

The frontend uses a custom `useAgentStream` hook that manages the full query lifecycle over Server-Sent Events:

```
connecting → session → classifying → classified → resolving → resolved
  → executing → widget_ready (x N) → executed → composing → done → result
```

The `widget_ready` event is the key design decision. As each backend tool completes, its result is streamed to the frontend, which immediately renders the corresponding widget. The user sees the company card appear in ~1 second, followed by the financial dashboard, map, timeline, and other widgets as they resolve -- rather than waiting for the entire pipeline to finish.

### Session Persistence

- localStorage persistence with a 24-hour resume window
- Data staleness probing on tab visibility change
- Automatic session retry on stale/expired errors (transparent to the user)
- Widget history navigation (browse back/forward through previous results)
- Entity accumulation across multi-turn interactions

---

## Widget Architecture

The Data Canvas renders the agent's dashboard through a `WidgetRenderer` dispatcher. 16 widget types:

| Widget | Description |
|---|---|
| `company_card` | Full company profile with status badge, legal form, address, activities, directors, health score. Supports compact mode. Embeds sector benchmark strip. |
| `financial_dashboard` | Multi-tab financial analysis: KPIs table, ratio spider chart, trend charts, bar charts. Auto-polls for pending data. |
| `compare_dashboard` | Side-by-side comparison with interactive metric selection and visual diff indicators. |
| `sector_financial_dashboard` | Sector-level overview: aggregate stats, rankings, distribution charts. |
| `table` | Sortable, filterable, paginated data table with search and CSV export. Clickable rows for entity navigation. |
| `map` | Interactive Leaflet map with clustered markers for establishment units. |
| `timeline` | Chronological event timeline with categorized entries. |
| `corporate_graph` | Interactive graph of corporate structure (React Flow + Dagre layout). |
| `network_graph` | Extended network visualization for cross-entity connections. |
| `insights_card` | Intelligence signals with severity-colored anomaly list and health score gauge. |
| `sector_benchmark_lite` | Compact sector positioning strip embedded in the company card. |
| `bar_chart` | Configurable bar chart (Recharts). |
| `pie_chart` | Pie chart (Recharts). |
| `scorecard` | Single-value KPI card. |
| `markdown` | Rendered markdown narrative (GFM). |

Widgets fetch data via `dataRef` identifiers through `/api/data/{ref}`, decoupling rendering from the SSE stream.

---

## SEO and Server Rendering

- Server-rendered company pages (`/company/[id]`) with JSON-LD structured data and Open Graph tags
- Dynamic `sitemap.xml` generated from all companies in the database
- Dynamic Open Graph image endpoint for social preview cards
- Canonical URLs and locale alternates for EN/FR/NL
- Deep-linkable analysis pages (`/analysis/[id]`)
- ISR (Incremental Static Regeneration) for company pages

---

## Internationalization

Full trilingual support (English, French, Dutch) across all UI strings, streaming stage labels, agent narratives, suggested actions, error messages, and disambiguation prompts. The backend agent responds in the user's detected or preferred language.

---

## Admin Dashboard (Backstage)

Password-protected admin interface with JWT auth:

| Page | What It Shows |
|---|---|
| Overview | LLM queue depth, cache stats, active sessions, shared state store |
| System Health | Auto-refreshing (5s) probes for NBB API, Python extractor, Gemini AI |
| Queries | Full query telemetry with search, intent filter, date range, pagination |
| Pipeline | Import run history with drill-down into individual runs |
| Analytics | Usage charts (daily queries, intent distribution, language breakdown) |
| Actions | Cache clearing, expired session purge, shared snapshot management |

---

**Previous:** [Agent Pipeline](agent-pipeline.md) | **Next:** [Infrastructure](infrastructure.md)
