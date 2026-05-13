# Architecture

This page covers the system topology and how the services connect.

For the full end-to-end walkthrough, start with the [README](../README.md) and follow the documentation links.

---

## System Diagram

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
                        └──┬──────────┬──────────┘      fallback SQL)
                           │          │
              ┌────────────▼──┐  ┌────▼──────────────┐
              │ PostgreSQL 16 │  │  Python Extractor  │
              │ KBO + Finance │  │  Flask + PyMuPDF   │
              │ Sessions      │  │  Vision API + LLM  │
              └───────────────┘  └───────────────────┘

         Background processes:
         ┌─────────────────────┐  ┌─────────────────────┐
         │  KBO/BCE Importer   │  │  Financial Worker    │
         │  (CLI, SAX parser)  │  │  (polling, refresh)  │
         └─────────────────────┘  └─────────────────────┘
```

## Service Inventory

| Service | Language | Role |
|---|---|---|
| **Frontend** | TypeScript / Next.js 16 | App shell, SSE streaming, 16 widget types, BFF proxy, SEO pages, admin dashboard |
| **Backend** | TypeScript / Fastify | Agent pipeline, tool execution, session management, financial engine, REST API |
| **PostgreSQL 16** | SQL | KBO enterprise data, financial data cache, sessions, import tracking, geocode cache |
| **Python Extractor** | Python / Flask | PDF balance sheet extraction via OCR and multimodal LLM |
| **KBO Importer** | TypeScript (CLI) | Full/daily/catchup imports of KBO XML dumps, reference code imports |
| **Financial Worker** | TypeScript | Background process polling for stale financial data and re-fetching |
| **Caddy** | Go | Reverse proxy with automatic HTTPS, security headers |

## Request Flow

A typical user query traverses the stack as follows:

1. The browser sends a POST to `/api/query/stream` on the Next.js frontend (Vercel Edge Runtime).
2. The Edge function injects the `x-api-key` header and proxies the request to the Fastify backend via Caddy.
3. Caddy terminates HTTPS and forwards to Fastify on port 3001.
4. Fastify opens an SSE stream back to the browser and begins the agent pipeline.
5. The agent classifies intent via a Gemini API call, resolves entities against PostgreSQL, plans tool calls deterministically, and executes tools in parallel.
6. As each tool completes, its result is stored in a shared state map (PostgreSQL-backed) and a `widget_ready` SSE event is sent to the browser.
7. The frontend renders each widget progressively as events arrive.
8. Once all tools complete, the dashboard is composed and the final `result` event is sent.

## Data Flow

```
                   ┌───────────────┐
SFTP (KBO XML) ───►│   Importer    │───► PostgreSQL (enterprise data)
                   └───────────────┘

                   ┌───────────────┐     ┌────────────────────┐
NBB CBSO API ─────►│  Fin. Engine  │────►│ Python Extractor   │
(XBRL + PDF)       │  (on-demand)  │     │ (PDF fallback)     │
                   └───────┬───────┘     └────────┬───────────┘
                           │                      │
                           ▼                      ▼
                   PostgreSQL (financial_years table)
```

---

**Next:** [Data Pipeline](data-pipeline.md)
