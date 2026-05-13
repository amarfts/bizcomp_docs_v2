# Tradeoffs and Limitations

This page documents the engineering constraints and compromises in the system. These are not bugs -- they are conscious decisions made under real-world constraints.

---

## Intent classification is a single LLM call with a long prompt

The classifier prompt is approximately 8KB of structured instructions covering 17 intent types, entity extraction rules, multilingual support, search filter parsing, and security directives. It works well for the Belgian business domain it was designed for, but it is inherently brittle in edge cases:

- Company names that look like person names (common in Belgium where companies are named after founders) require a post-classification heuristic layer that checks for commercial suffixes
- The follow-up vs. data_question vs. new-query distinction relies on the LLM correctly interpreting conversational context, which occasionally fails on ambiguous phrasing
- Adding a new intent type requires careful prompt engineering to avoid regressions in existing intent accuracy

The tradeoff was deliberate: a single-call classifier is ~500ms faster than a multi-stage classification pipeline, and the post-hoc heuristic layer catches the most common misclassifications.

---

## PDF extraction is a 3-layer cascade because no single approach works

Belgian balance sheets come in dozens of formats: full-schema XBRL, abbreviated XBRL, micro-entity schemas, scanned PDFs, digitally-generated PDFs with inconsistent table layouts, bilingual documents (French/Dutch columns), and occasionally German. No single extraction method handles all of them:

- XBRL parsing is fast and accurate but only available for ~60-70% of filings
- Vision API OCR with coordinate-based extraction works for standard layouts but fails on non-standard column arrangements or when the "current year" / "previous year" headers are missing
- The Gemini multimodal fallback handles non-standard layouts but is slower and more expensive

The cascade approach means that the cheapest, fastest method is tried first, with more expensive fallbacks only triggered when validation fails. The validation gate checks for the presence of critical fields (total assets or net profit) before accepting a result.

---

## Financial data is hidden until the 5-year window is complete

The system fetches 5 years of financial history per company. Data is not shown to the user until all 5 years have been processed (or confirmed as having no filing). This prevents users from seeing incomplete dashboards where some years are missing mid-sequence. The downside: recently-founded companies that have only 1-2 years of filings may show "data is being retrieved" while the system confirms that the remaining years have no filings.

---

## The deterministic pipeline eliminates the LLM for ~90% of queries -- but not all

For standard intents (company overview, financials, map, timeline, risk, deep dive, comparison, search, sector benchmark), the entire dashboard is assembled in TypeScript with no LLM call after classification. This saves 1-3 seconds per query and makes the output fully reproducible.

However, aggregate queries ("How many companies went bankrupt in Wallonia last year?") and custom queries that do not map to a predefined tool require the LLM fallback loop, where Gemini generates SQL queries and iterates. This path is slower (~5-15s), less predictable, and can fail if the generated SQL is invalid. A 2-consecutive-error circuit breaker stops the loop and forces a best-effort summary.

---

## Entity resolution depends on PostgreSQL full-text search

Company name resolution uses a PostgreSQL materialized view with a GIN-indexed `tsvector` column. This is fast (~5-20ms per lookup) but has inherent limitations:

- Very common name fragments ("Transport", "Services", "Group") return many results and trigger disambiguation
- The resolver truncates long search strings to 4 words to avoid expensive multi-token GIN queries, which can lose precision for companies with long names
- Juridical form suffixes (BV, NV, SRL, SA) are stripped before search because they are not part of the canonical name in the database, but this means "Eforge BV" and "Eforge NV" resolve to the same search query

---

## No horizontal scaling strategy for sessions

Sessions are stored in PostgreSQL. There is no Redis layer, no distributed session store, and no session affinity. This works for the current traffic level but would need to change for multi-instance deployment. The session store does implement expiry and cleanup (via the admin dashboard), but the architecture assumes a single backend instance.

---

## The frontend is a monolith

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

---

**Previous:** [Infrastructure](infrastructure.md) | **Back to:** [README](../README.md)
