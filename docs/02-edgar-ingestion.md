# 02 — EDGAR ingestion

*Status: **live.** Filing-index ingestion, 8-K Item 2.02 earnings events, XBRL fundamentals
(`companyfacts`), and **daily incremental** re-ingest of CIKs that filed since the last run are
implemented and running against production. Pre-XBRL text extraction is a separate tier
([`09-pre-xbrl-fundamentals.md`](09-pre-xbrl-fundamentals.md)).*

SEC EDGAR is ptdata's foundational, public-domain source. This document specifies how we
ingest it and how versioning/linking work.

## Implemented — filing index

- The structured submissions feed (`https://data.sec.gov/submissions/CIK##########.json`) is
  fetched with a descriptive SEC-compliant User-Agent, parsed into filing records, and
  upserted into `filing` keyed by `accession_number` — **append-only and idempotent**
  (re-ingesting an accession is a no-op). Each company is get-or-created by CIK; every run is
  recorded in `ingestion_run`.
- Captured per filing: form type, filing date, exact acceptance timestamp (tz-aware), period
  of report, primary-document URL, and 8-K items.
- **Deep history:** both the `filings.recent` block and the older `filings.files[]` pages are
  ingested, so filing coverage spans a company's full EDGAR history. Company tickers from the
  submissions feed are linked to securities (resolving ticker → company). `amends_accession`
  linkage (restatement chains) is still resolved by a later step, not at index ingest.

## Implemented — earnings announcement events (8-K Item 2.02)

- Each ingested 8-K carrying **Item 2.02** ("Results of Operations and Financial Condition")
  becomes an `earnings_event`, keyed by its 8-K accession (idempotent), capturing the exact
  `sec_acceptance_ts` from EDGAR.
- **Still thin by design:** market `dissemination_ts` / wire time is a separate sourcing effort
  (many events still carry `dissemination_confidence = 'unknown'`). Fiscal-period attribution on
  earnings events is enriched where possible from fundamentals context; see
  [`06-earnings-events.md`](06-earnings-events.md) and [`08-known-limitations.md`](08-known-limitations.md).

## Implemented — XBRL fundamentals + daily incremental

- **XBRL fundamentals** from `companyfacts` → `financial_fact` with `provenance='xbrl'`,
  bitemporal supersession, and methodology versioning ([`03-fundamentals.md`](03-fundamentals.md)).
- **Daily incremental:** EDGAR daily master index → re-ingest only CIKs that filed forms of
  interest since the last watermark (`edgar:incremental` in `ingestion_run`). Temporary SEC 403s
  for not-yet-published indexes do not permanently skip those days.
- **Pre-XBRL text parse** is offline/batch and gap-fills only
  ([`09-pre-xbrl-fundamentals.md`](09-pre-xbrl-fundamentals.md)).

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

## Open documentation gaps (not product blockers)

SEC fair-access / User-Agent practice; full concept→tag map; fiscal-year-end change edge cases;
dissemination-time sourcing if/when wire times ship.
