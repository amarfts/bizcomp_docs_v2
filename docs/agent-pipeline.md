# Agent Pipeline

Every user query follows a deterministic pipeline. The design philosophy is to avoid LLM calls wherever possible -- the LLM is used for classification and for queries that cannot be planned deterministically, but the majority of the pipeline is pure TypeScript logic.

---

## Pipeline Overview

```
User Message
    │
    ▼
┌──────────────────────────┐
│ Fast Path Check          │  regex detects enterprise number in message
│                          │  → skip classification, go straight to tools
└────────────┬─────────────┘
             │ (no match)
             ▼
┌──────────────────────────┐
│ Intent Classification    │  single Gemini call → 17 intents + entities
│                          │  parallel: speculatively resolve active companies
└────────────┬─────────────┘
             │
             ├── greeting         → canned response (no LLM)
             ├── clarification    → single LLM response
             ├── data_question    → conversational answer about on-screen data
             │
             ▼
┌──────────────────────────┐
│ Entity Resolution        │  PostgreSQL full-text search (GIN index)
│                          │  fuzzy matching, confidence scoring
│                          │  disambiguation when confidence is low
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│ Query Planning           │  deterministic TypeScript: intent + entities → tools
│                          │  no LLM involved
└────────────┬─────────────┘
             │
             ├── aggregate/custom → LLM fallback loop (SQL generation)
             │
             ▼
┌──────────────────────────┐
│ Parallel Tool Execution  │  13 tools, concurrent execution
│                          │  session-level result caching
│                          │  progressive SSE events
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│ Dashboard Composition    │  deterministic builders (no LLM for ~90% of queries)
│                          │  localized summary narratives
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│ Suggested Actions        │  context-aware follow-up suggestions
│                          │  filtered, localized (EN/FR/NL)
└──────────────────────────┘
```

---

## Fast Path

When the user message contains a Belgian enterprise number (e.g., from clicking a search result or pasting a number), the system skips the LLM classifier entirely. It:

1. Parses the enterprise number with regex
2. Infers the intent from keyword analysis (financial terms, map terms, risk terms, etc.)
3. Routes directly to entity resolution and the deterministic pipeline

This saves ~500ms-1.5s per query by eliminating the classification LLM call.

---

## Intent Classification

A single Gemini call with a structured system prompt classifies the user's message into one of 17 intent types and extracts structured entities.

### Intent Types

| Intent | Description | Pipeline Path |
|---|---|---|
| `company_overview` | General company summary | Deterministic |
| `company_financials` | Financial data request | Deterministic |
| `company_map` | Establishment locations | Deterministic |
| `company_timeline` | Historical events | Deterministic |
| `company_risk` | Anomaly/health analysis | Deterministic |
| `company_deep_dive` | Full multi-domain analysis | Deterministic |
| `company_group` | Corporate structure | Deterministic |
| `company_data_lookup` | Specific metric query (e.g., "what is their FTE?") | Deterministic |
| `compare_companies` | Side-by-side comparison | Deterministic |
| `person_investigation` | Person mandate search | Deterministic |
| `company_search` | Filtered company search | Deterministic |
| `sector_benchmark` | Sector-level analysis | Deterministic |
| `network_analysis` | Extended corporate network | Deterministic |
| `aggregate_query` | Custom aggregate/statistical question | LLM Fallback |
| `custom_query` | Free-form database query | LLM Fallback |
| `greeting` | Greeting / small talk | Canned response |
| `data_question` | Question about on-screen data | Conversational |

### Extracted Entities

The classifier extracts structured entities from the user's message:

- Company names (with support for partial names, abbreviations, and commercial names)
- Person names
- NACEBEL sector codes
- Financial metrics (revenue, profit, FTE, equity, margin, etc.)
- Search filters (region, status, age range, FTE range, revenue range, etc.)
- Language detection

### Post-Classification Heuristics

A TypeScript heuristic layer runs after classification to handle Belgian-specific edge cases:

- Dutch family names that are also company names (Van Oirschot, De Smedt) are disambiguated using commercial suffix detection (BV, NV, SRL, SA, etc.)
- If the LLM classifies a message as `person_investigation` but a commercial suffix is present, it is reclassified to a company intent
- Follow-up vs. new-query disambiguation is refined using session context

The classifier operates at temperature 0.0 for determinism.

---

## Entity Resolution

Company names extracted by the classifier are resolved to enterprise numbers via PostgreSQL full-text search.

### Resolution Steps

1. **Normalize input** -- strip juridical form suffixes (BV, NV, SRL, SA), truncate long queries to 4 meaningful words
2. **Check for embedded enterprise numbers** -- detect enterprise numbers in mixed strings like "show me 0417497106"
3. **Full-text search** -- query the GIN-indexed materialized view with ranked results
4. **Confidence scoring** -- score results by relevance; return disambiguation tables when confidence is low
5. **Cache lookup** -- LRU cache (500 entries, 10-minute TTL) avoids redundant lookups

### Speculative Resolution

On follow-up queries, entity resolution runs speculatively in parallel with intent classification. The system resolves the currently active company names while the classifier is still processing, then merges the results. This hides the entity resolution latency behind the classification latency.

---

## Query Planning

A pure TypeScript function maps each intent type to a set of tool calls. No LLM is involved. The planner:

- Maps 14 intent types to specific tool combinations
- Handles follow-up deduplication (skips the company card if the user already has one on screen for the same company)
- Forwards search filters for multi-turn search refinements
- Detects when a query requires the LLM fallback loop (aggregate and custom queries)
- Determines whether to include a full or compact company card based on context

### Example Mappings

| Intent | Tool Calls |
|---|---|
| `company_overview` | `get_company_card` + `company_anomalies` (+ `get_financials` if metrics requested) |
| `company_financials` | `get_company_card` (compact) + `get_financials` |
| `company_deep_dive` | `get_company_card` + `get_financials` + `map_establishments` + `company_timeline` + `company_anomalies` |
| `compare_companies` | `get_financials` (per company) + `compare_companies` |
| `company_search` | `search_companies` (with all extracted filters) |
| `aggregate_query` | (empty) -- routed to LLM fallback loop |

---

## Tool Execution

13 domain-specific tools execute in parallel via `Promise.allSettled`:

| Tool | Data Domain | Description |
|---|---|---|
| `get_company_card` | Profile | Full company profile: status, legal form, addresses, activities, directors, capital, health score |
| `get_financials` | Financial | 5-year KPI history with ratios; triggers on-demand NBB fetch if data is missing |
| `map_establishments` | Geographic | All establishment units with geocoded addresses (Google Maps API with database-level cache) |
| `company_timeline` | Historical | Chronological history: name changes, address changes, activity changes, status changes |
| `company_anomalies` | Intelligence | Health score calculation, anomaly detection (director churn, rapid address changes, negative equity, etc.) |
| `compare_companies` | Comparison | Side-by-side comparison of 2+ companies across all financial and structural metrics |
| `find_related_companies` | Corporate group | Parent/subsidiary relationships via shared directors and mandates |
| `network_analysis` | Network | Extended graph of corporate connections across multiple hops |
| `investigate_person` | Person | All corporate mandates held by a person across Belgian companies |
| `search_companies` | Search | Filtered company search with 15+ dimensions (sector, region, status, age, FTE, revenue, profit, equity, capital, director count, etc.) |
| `sector_financials` | Sector | Aggregate sector benchmarks: median revenue, margin distribution, FTE totals, company rankings |
| `sector_snapshot` | Sector | Quick structural stats: active count, bankruptcies, average age, top locations |
| `sql_query` | Custom | Raw SQL execution against the read-only database (fallback path only) |

Each tool result is:
1. Stored in a shared state map backed by PostgreSQL
2. Assigned a `dataRef` identifier for frontend retrieval
3. Streamed to the frontend as a `widget_ready` SSE event as soon as it completes

This enables progressive rendering -- the user sees widgets appear one by one as tools finish, rather than waiting for the entire pipeline.

### Session-Level Caching

Tool results are cached at the session level. If the same tool with the same arguments is called again within the same session (e.g., a follow-up query about the same company), the cached result is returned immediately without re-execution.

---

## Dashboard Composition

### Deterministic Builders (~90% of queries)

For standard intents, the dashboard is assembled entirely in TypeScript with no LLM call. The deterministic builders:

- Map tool results to widget types (`company_card`, `financial_dashboard`, `map`, `timeline`, `insights_card`, etc.)
- Generate localized summary narratives from the actual data
- Include contextual information: anomaly counts, health scores, risk levels, financial trend analysis
- Handle edge cases: pending financial data, disambiguation tables, partial tool failures

**Summary narrative examples** (generated deterministically from data, not by an LLM):

- "Revenue at [Company] grew 12% over 4 years to EUR1.2B, net margins compressed from 3.2% to 2.8%."
- "High risk -- 5 anomalies detected (health score: 42/100). Director churn: 3 changes in 12 months."
- "[Company] has 14 active establishments across Belgium."

### LLM Fallback Composer

For aggregate and custom queries that use the SQL tool-calling loop, the LLM generates the final dashboard JSON directly. The fallback loop:

1. Sends the user query with the database schema to Gemini
2. Gemini generates a `sql_query` tool call
3. The system executes the SQL against a read-only database connection
4. Results are returned to Gemini, which either generates another query or produces a final dashboard
5. Iterates up to 6 times
6. A 2-consecutive-error circuit breaker stops the loop if SQL generation is consistently invalid

---

## Session Management

Multi-turn conversations are managed via server-side sessions stored in PostgreSQL.

### Session State

| Field | Purpose |
|---|---|
| Conversation history | Full user/agent turn pairs |
| Active entities | Company names, enterprise numbers, persons, NACEBEL codes |
| Active search filters | Preserved across search refinements |
| Tool result cache | Avoids re-fetching identical results within a session |
| Last data domains | Tracks what is already on screen to avoid redundant widgets |
| Previous widget types | Used to determine compact vs. full company card rendering |

### Follow-Up Resolution

- **Pronoun references** ("show me their finances") inherit the active company from session context
- **Search refinements** ("now only in Antwerp") merge the new filter with the previous search's full filter set
- **Domain follow-ups** ("what about the map?") reuse the active entity and switch only the intent

---

## Suggested Actions

After each response, the system generates context-aware follow-up suggestions. These are deterministic TypeScript functions that:

- Suggest relevant next steps based on the current intent and data domains already shown
- Filter out actions the user has already performed in the session
- Localize all suggestions into EN/FR/NL
- Include entity-specific actions (e.g., "Compare with [specific competitor]" when competitors are detected in the corporate group)

---

**Previous:** [Data Pipeline](data-pipeline.md) | **Next:** [Frontend](frontend.md)
