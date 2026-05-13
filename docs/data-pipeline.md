# Data Pipeline

This page covers how data enters the system: KBO/BCE enterprise imports, NBB financial data acquisition, and the PDF extraction cascade.

---

## KBO/BCE Enterprise Data

The Belgian Crossroads Bank for Enterprises (KBO/BCE) publishes the complete enterprise registry as XML dumps, available via SFTP from `ftps.economie.fgov.be`. The dataset contains:

- ~3 million enterprise records (active, stopped, bankrupt, in judicial reorganization)
- Denominations (official names, commercial names, abbreviations) in French, Dutch, and German
- Registered addresses with municipality, postal code, and street-level detail
- NACEBEL activity codes (Belgian extension of NACE Rev. 2)
- Juridical forms, establishment units, contact information
- Mandate holders (directors, administrators, managers) with start/end dates

### Import Architecture

The importer uses SAX-based streaming XML parsers to process the full dump without loading the entire file into memory. This is necessary because the full XML is several gigabytes. The import pipeline supports five modes:

| Mode | Purpose |
|---|---|
| `migrate` | Runs the database schema migration (DDL) |
| `codes` | Imports reference codes (activity codes, juridical forms, status codes) |
| `full` | Full enterprise import from the complete XML dump |
| `daily` | Incremental update from the daily delta XML |
| `catchup` | Replays missed daily updates to close gaps |

The importer also supports automatic SFTP download of the latest ZIP archive, with host fingerprint verification.

### Search Infrastructure

A materialized view (`enterprise_search`) with a GIN-indexed `tsvector` column supports fast full-text company search with ranked results across canonical names, commercial names, and abbreviations. This is the backing store for the agent's entity resolution stage.

---

## NBB Financial Data

Financial data comes from the National Bank of Belgium (NBB) via their CBSO API. Two acquisition strategies run in parallel:

### Bulk XBRL Import

A CLI process downloads batch ZIP archives from the NBB for every business day in the last N years. Each ZIP contains all XBRL filings deposited on that date. The system:

1. Parses the XBRL rubrics (Belgian accounting plan codes: 70 for revenue, 9904 for net profit, 10/15 for equity, etc.)
2. Transforms them into a normalized set of KPIs (revenue, net profit, total assets, equity, FTE headcount, gross margin, operating profit, depreciation, etc.)
3. Calculates derived ratios (operating margin, debt-to-equity, current ratio, equity ratio, net profit margin)
4. Upserts the results into the `financial_years` table

The process is resumable via a `financial_year_coverage` table that tracks per-company, per-year completeness with statuses: `complete`, `no_filing`, `pending_bulk`, `needs_on_demand_enrichment`, `error`.

### On-Demand Per-Company Fetch

When a user queries a company that has no local financial data, the system triggers a real-time fetch. This follows a waterfall:

1. Fetch filing references from the NBB API
2. Attempt XBRL download and parse
3. If XBRL is unavailable or incomplete (common for older filings or micro-entity schemas), fall back to PDF extraction

The financial engine tracks which companies are currently being fetched to avoid duplicate work across concurrent requests. Results are stored and cached for subsequent queries.

---

## PDF Extraction Cascade

The Python extractor service implements a 3-layer cascade for extracting structured financial data from Belgian balance sheet PDFs:

```
PDF Input
    │
    ▼
┌──────────────────────────────┐
│ Layer 1: Fast Path           │
│ Text-layer keyword search    │  ~2-5s
│ OCR only matched pages       │
│ Coordinate-based extraction  │
└────────────┬─────────────────┘
             │ validation fails?
             ▼
┌──────────────────────────────┐
│ Layer 2: OCR Fallback        │
│ Full OCR of all pages        │  ~10-30s
│ Page classification          │
│ Coordinate-based extraction  │
└────────────┬─────────────────┘
             │ validation fails?
             ▼
┌──────────────────────────────┐
│ Layer 3: LLM Fallback        │
│ Render pages as images       │  ~15-45s
│ Send to Gemini in chunks     │
│ Parse structured JSON output │
└──────────────────────────────┘
```

### Layer 1: Fast Path

The PDF's text layer is scanned for keywords that identify specific page types (balance sheet assets, liabilities, income statement, social balance). Only matching pages are rendered and sent to the Google Vision API for OCR. Values are extracted by:

1. Detecting column zones -- locating the "current year" (BOEKJAAR/EXERCICE) and "previous year" (VORIG BOEKJAAR/EXERCICE PRECEDENT) headers
2. Finding accounting code labels (e.g., "70", "9904", "10/15") by text matching
3. Extracting numeric values in the same row that fall within the detected column zones
4. Stitching multi-word numbers (e.g., "1" + "." + "234" + "." + "567") into complete values

### Layer 2: OCR Fallback

If the fast path fails validation (no total assets or net profit found), the system renders every page at 300 DPI and sends them all through the Vision API. Pages are classified by their OCR text content. A fail-fast mechanism aborts after the first 10 pages if no financial content is detected, avoiding unnecessary API costs.

The OCR fallback runs in two phases:
- Phase 1: Render and OCR the first 10 pages. If no classified pages are found, abort.
- Phase 2: Only if Phase 1 found financial content, render and OCR the remaining pages.

### Layer 3: LLM Fallback

When coordinate-based extraction fails entirely (non-standard table layouts, unusual column arrangements, scanned documents with poor OCR quality), the system falls back to Gemini's multimodal capabilities:

1. Pages identified during OCR classification (or neighboring pages) are selected
2. Pages are rendered at configurable DPI and grouped into chunks
3. Each chunk is sent to Gemini with a structured extraction prompt
4. Results are merged across chunks, with conflict resolution for overlapping data
5. LLM-extracted field names are remapped to the standard database schema

The LLM path includes retry logic with exponential backoff for API rate limits and transient failures.

### Validation

Each layer validates its output before accepting it. The validation gate checks for the presence of critical fields -- specifically `total_assets` or `net_profit`. If neither is present, the result is considered invalid and the next layer is tried.

After extraction, a cleaning and normalization step:
- Converts European number formats (1.234.567,89) to standard floats
- Applies fallback chains for metrics with multiple possible accounting codes (e.g., FTE can appear under code 9087, 9086, or the VKT equivalent)
- Calculates derived values (gross margin from revenue minus costs when the direct code is missing)

---

## Financial Worker

A background process (`financial-worker`) polls the database every 5 seconds for companies with stale financial data. It processes them in configurable batches (default: 20 companies, 4 concurrent fetches) with rate limiting to respect NBB API quotas. This keeps the dataset current without requiring user-triggered refreshes.

---

**Previous:** [Architecture](architecture.md) | **Next:** [Agent Pipeline](agent-pipeline.md)
