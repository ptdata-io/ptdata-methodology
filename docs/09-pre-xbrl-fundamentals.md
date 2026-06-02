# 09 — Pre-XBRL fundamentals (the parsed tier)

SEC structured XBRL (`companyfacts`) begins ~2009. For periods before that — and for companies
that died before it (Enron, WorldCom, Lehman) — fundamentals can only be obtained by reading the
filing text. ptdata serves a small, **clearly-labeled, lower-confidence tier** of such values,
extracted from the original filings. It is *additive* to the XBRL core and never replaces it.

## What it is
- **Metrics:** revenue and net income only (the same logical metrics as the XBRL tier).
- **Scope (Phase 1):** a *curated* set of high-value companies — notable defunct issuers plus a
  few S&P stalwarts for boundary continuity — not the full pre-XBRL universe. Annual (10-K) is
  the default; specific flagship quarters (10-Q) are included where they back a golden case.
- **Labeling:** every value carries `provenance` and `extraction_confidence` (below). XBRL values
  are `provenance="xbrl"` with no extraction confidence; parsed values are `provenance="parsed"`.

## How values are extracted
The financial-statements region of the primary 10-K/10-Q is isolated, then revenue + net income
are obtained from up to three independent signals and reconciled:
1. **Deterministic table parse** — reads the statement's scale ("in millions/thousands"), the
   year-column header (so column *order* is irrelevant), and the exact line labels ("Total
   revenues" / bottom-line "Net Income", not "Net income before…"). For quarterly statements that
   show both three-month and nine-month columns, the three-month (the quarter) is taken.
2. **Claude-assisted extraction** — the located statement text is read by an LLM that returns the
   same fields as structured JSON (offline/batch only — never in the live service).
3. **MD&A prose** — the headline figures restated in the narrative ("net income for 2000 was
   $979 million…") as an independent check.

## Confidence tiers
`extraction_confidence` reflects agreement across those signals:
- **verified** — the value was confirmed against the primary filing during ingestion (against the
  authoritative source document). Only verified values back golden cases; never assigned
  automatically by the extractor.
- **high** — ≥2 independent signals agree.
- **medium** — a single signal produced the value.
- **low** — signals conflicted; held for review, not served.

## Integration rules (point-in-time + lineage preserved)
- **Gap-fill only — XBRL always wins.** A parsed fact is written only when no XBRL fact exists for
  that (company, concept, fiscal_year, fiscal_period). The two tiers never collide.
- **Bitemporal + lineage unchanged:** `knowledge_from` = the filing's acceptance time;
  `source_accession` = the originating 10-K/10-Q. A point-in-time query returns the value known on
  the as-of date, exactly as for XBRL facts.
- **Same query surface:** parsed facts are returned by `get_fundamentals` alongside XBRL facts,
  distinguished only by the `provenance` / `extraction_confidence` fields.

## Current coverage (measured)
- **Parsed tier:** 3 companies, 6 facts, all `verified`:
  - Enron (CIK 0001024401) Q3 2001 — revenue $46,877M, net income $(644)M (accn 0000950129-01-504218).
  - WorldCom/MCI (CIK 0000723527) Q1 2002 — *consolidated* revenue $8,120M, net income $172M
    (accn 0000912057-02-020812; the original pre-fraud-disclosure figures, not the MCI-group subtotal).
  - Exxon Mobil (CIK 0000034088) FY2008 — revenue $459,579M (sales-and-other-operating-revenue
    basis, matching the us-gaap:Revenues concept used from FY2009), net income $45,220M
    (accn 0001193125-09-040966). A continuity case: the parsed FY2008 sits directly below XOM's
    XBRL FY2009 ($301.5B) on the same basis — no artificial discontinuity at the boundary.
- The Enron + WorldCom facts back the `enron_2001_q3_10q` and `worldcom_2002_10q` correctness cases.

**On the extraction mechanism:** interactively (in a Claude Code session) the Claude signal is
produced by a subagent on the operator's subscription — no API key. The standalone batch tool
(`claude_extract`, for headless/cron runs) is the only path that needs `ANTHROPIC_API_KEY`. Note
the deterministic table parser is line-based and does **not** handle HTML filings where each cell
renders on its own line (e.g. XOM FY2008) — there the subagent (or `claude_extract`) is the
extraction signal, cross-checked against the primary text.

## Known limitations of this tier
- **Curated, not comprehensive.** Only the curated companies/periods are covered; absence of a
  pre-2009 value is expected, not a guarantee of non-existence.
- **Extraction has an error profile.** Unlike XBRL (machine-tagged at the source), these values are
  read from unstructured text. Only `verified` values are served today; the broader curated run
  (Claude-cross-checked, `high`/`medium`) is gated behind review before exposure.
- **`valid_range` is a calendar-year approximation** for annual parsed facts (the authoritative key
  is `(fiscal_year, fiscal_period)`; the date range is informational). Non-calendar fiscal years
  are therefore approximate in `valid_range` only.
- **No restatement chains within the parsed tier** (rare pre-XBRL restatements are not modeled as
  bitemporal versions here; the point-in-time guarantee holds against the original filing).
- **No Compustat/vendor data** — this tier is parsed from public-domain EDGAR filings only, so it
  remains redistributable. See [`08-known-limitations.md`](08-known-limitations.md).
