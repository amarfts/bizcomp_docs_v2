# BizComp

**AI-powered intelligence platform for the Belgian corporate registry.**

BizComp ingests the full KBO/BCE enterprise database (~3 million entities) and the NBB financial filing archive, then exposes this data through a conversational agent that classifies user intent, resolves company references, executes domain-specific tools in parallel, and assembles interactive dashboards -- all streamed to the browser in real time via Server-Sent Events.

The source code is private. This repository documents the system architecture, design decisions, and engineering tradeoffs.

---

## System Overview

```
User Browser
      │
      ▼
Vercel (Next.js 16) ─── Edge Runtime, SSE Streaming, 16 Widget Types
      │
      ▼
Caddy ─── Automatic HTTPS, Security Headers
      │
      ▼
Fastify API Server ─── Agent Pipeline, Session Management ──── Gemini API
      │         │
      ▼         ▼
PostgreSQL    Python Extractor ─── Vision API, Multimodal LLM
```

### At a Glance

| Dimension | Value |
|---|---|
| Enterprise records | ~3 million |
| Denomination records | ~15 million |
| Activity records | ~5 million |
| Financial years tracked | 5 per company |
| Agent intent types | 17 |
| Agent tools | 13 |
| Widget types | 16 |
| Search filter dimensions | 15+ |
| Languages | English, French, Dutch |
| Deterministic path latency | 1.5--4s |
| Fallback path latency | 5--15s |

---

## Documentation

| Document | Description |
|---|---|
| [Architecture](docs/architecture.md) | System topology, service inventory, how everything connects |
| [Data Pipeline](docs/data-pipeline.md) | KBO/BCE imports, NBB financial data, PDF extraction cascade, background workers |
| [Agent Pipeline](docs/agent-pipeline.md) | Intent classification, entity resolution, query planning, tool execution, dashboard composition, session management |
| [Frontend](docs/frontend.md) | SSE streaming, progressive rendering, 16 widget types, SEO, internationalization, admin dashboard |
| [Infrastructure](docs/infrastructure.md) | Docker Compose stack, PostgreSQL tuning, Vercel Edge deployment, security |
| [Tradeoffs](docs/tradeoffs.md) | Engineering constraints, limitations, and conscious design compromises |

---

## Tech Stack

### Backend

| Component | Technology |
|---|---|
| API Server | TypeScript, Fastify |
| Database | PostgreSQL 16 |
| AI/LLM | Google Gemini (classification, composition, fallback SQL) |
| PDF Extraction | Python, Flask, PyMuPDF, Google Vision API, Gemini multimodal |
| Data Import | SAX-based XML stream parsers, XBRL parser |
| Reverse Proxy | Caddy |
| Containerization | Docker Compose |

### Frontend

| Component | Technology |
|---|---|
| Framework | Next.js 16 (App Router) |
| Language | TypeScript 5, React 19 |
| Styling | Tailwind CSS 4 |
| Charts | Recharts 3 |
| Maps | Leaflet, React-Leaflet |
| Network Graphs | React Flow (@xyflow/react), Dagre |
| Internationalization | next-intl 4 |
| Deployment | Vercel (Edge Runtime) |

---

## Repository Structure

```
docs/
  architecture.md       System topology and service inventory
  data-pipeline.md      Data ingestion and financial extraction
  agent-pipeline.md     AI agent pipeline design
  frontend.md           Frontend architecture and widget system
  infrastructure.md     Deployment, security, and testing
  tradeoffs.md          Limitations and design compromises
```
