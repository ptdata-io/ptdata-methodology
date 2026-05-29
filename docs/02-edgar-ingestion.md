# 02 — EDGAR ingestion

*Scaffold — to be completed alongside ingestion implementation (build step 6a/6b).*

SEC EDGAR is ptdata's foundational, public-domain source. This document will specify how we
ingest it and how versioning/linking work.

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
