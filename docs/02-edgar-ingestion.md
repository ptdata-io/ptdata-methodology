# 02 — EDGAR ingestion

*Status: filing-index ingestion (6a) and 8-K Item 2.02 earnings events (6b) implemented;
fundamentals (6c) pending. This document is filled in as those land.*

SEC EDGAR is ptdata's foundational, public-domain source. This document specifies how we
ingest it and how versioning/linking work.

## Implemented (6a)

- The structured submissions feed (`https://data.sec.gov/submissions/CIK##########.json`) is
  fetched with a descriptive SEC-compliant User-Agent, parsed into filing records, and
  upserted into `filing` keyed by `accession_number` — **append-only and idempotent**
  (re-ingesting an accession is a no-op). Each company is get-or-created by CIK; every run is
  recorded in `ingestion_run`.
- Captured per filing: form type, filing date, exact acceptance timestamp (tz-aware), period
  of report, primary-document URL, and 8-K items.
- **Coverage caveat:** only the `filings.recent` block (~1,000 latest filings per company) is
  ingested so far. Full 2010-present depth requires paging `filings.files[]` — a tracked
  follow-up. `amends_accession` linkage (restatement chains) is resolved by a later step, not
  at index ingest.

## Implemented (6b — earnings announcement events)

- Each ingested 8-K carrying **Item 2.02** ("Results of Operations and Financial Condition")
  becomes an `earnings_event`, keyed by its 8-K accession (idempotent), capturing the exact
  `sec_acceptance_ts` from EDGAR.
- **Deferred by design:** fiscal-period attribution (`fiscal_year`/`fiscal_period`) and the
  market `dissemination_ts`. An 8-K's `period_of_report` is the announcement date, not the
  fiscal-period end, so attribution is enriched authoritatively in 6c (from the as-reported
  parse); dissemination/wire time is a separate sourcing effort. Until then events carry
  `dissemination_confidence = 'unknown'` and null fiscal attribution — see
  [`06-earnings-events.md`](06-earnings-events.md) and [`08-known-limitations.md`](08-known-limitations.md).

## Scope

- **Filing index.** The daily/full EDGAR index → one `filing` row per submission, keyed by
  `accession_number`, capturing `form_type`, `filing_date`, `acceptance_datetime` (exact, to
  the second, ET), `period_of_report`, and document URLs.
- **8-K Item 2.02 parsing.** Identify earnings-announcement 8-Ks and extract the acceptance
  timestamp and the press-release exhibit → `earnings_event.sec_acceptance_ts` and the
  as-reported `financial_fact` rows sourced from that accession.
- **Fundamentals.** As-reported financial values from 10-K/10-Q (and the 8-K exhibit for
  headline figures), normalized via `financial_concept`.

## Versioning & linking (the rules this doc will make precise)

- Every filing is immutable once ingested; re-ingestion is idempotent.
- Amendments/restatements link to the amended filing via `amends_accession`, forming the
  chain that powers as-reported-vs-restated queries ([`03`](03-fundamentals.md)).
- Each ingestion is recorded in `ingestion_run` for audit/lineage.

## To document here

Rate-limit / fair-access compliance with SEC; XBRL vs. exhibit parsing strategy; concept
normalization map; handling of fiscal-year-end changes; dissemination-time sourcing inputs.
