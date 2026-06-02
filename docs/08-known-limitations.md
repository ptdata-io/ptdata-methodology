# 08 — Known limitations

The moat of this product is honesty about what it does and does not guarantee. This document
is the canonical statement of current limitations and coverage gaps. It is updated whenever
behavior changes. If you find a gap not listed here, that is a bug in this document — please
open an issue on [ptdata-correctness](https://github.com/ptdata/ptdata-correctness).

## Status

**Live — full-universe backfill complete (2026-05-31); pre-XBRL parsed tier added (2026-06-02).**
The figures below are *measured against the production database*, not targets. Coverage is gated by
the public correctness suite (currently `7 passed, 6 skipped, 0 failed`; skipped cases mark
out-of-scope behavior, never a silent pass).

| Measure | Value |
|---|---|
| Companies (EDGAR universe + curated pre-XBRL) | 7,994 |
| — with fundamentals (us-gaap XBRL) | 5,975 (74.8%) |
| — filings-only (no us-gaap XBRL — funds/trusts/foreign/small filers) | 2,017 (25.2%) |
| Filings ingested (all form types, full history) | ~6.78M |
| Earnings events (8-K Item 2.02) | 224,310 |
| Financial facts — XBRL tier (bitemporal versions) | 547,532 |
| Financial facts — parsed pre-XBRL tier (`provenance='parsed'`) | 14 (3 companies) |
| Fundamentals period floor — XBRL tier | FY2009 |
| S&P 500 membership (distinct securities, historical) | 849 |
| Database size | ~2.3 GB |

Metrics ingested: **revenue** and **net income** only (logical metrics resolved across us-gaap
tags; income-statement / duration concepts). Fundamentals are kept fresh by a daily incremental
ingestion of EDGAR's daily index (re-ingesting only CIKs that filed since the last run).

**Pre-XBRL parsed tier (`provenance='parsed'`).** Below the FY2009 XBRL floor, a small *curated*,
clearly-labeled, lower-confidence tier is parsed from filing text — currently Enron Q3 2001 and
WorldCom Q1 2002 (`verified`; the original pre-collapse figures), plus Exxon Mobil FY2004–2008 (a
continuity series below its XBRL FY2009; net income `high`/`verified`, revenue `medium`). Gap-fill
only (XBRL always wins); never replaces structured data. Full methodology + limitations:
[`09-pre-xbrl-fundamentals.md`](09-pre-xbrl-fundamentals.md).

## By design (not gaps — deliberate scope decisions)

- **No analyst consensus.** ptdata does not carry analyst EPS/revenue consensus, current or
  historical. Genuine point-in-time consensus is not redistributable at an accessible price;
  a cheap reconstruction would reintroduce lookahead bias. We serve the **options-implied
  expected move** instead, which is a different quantity and is never labeled "consensus."
  Rationale: [`07-options-implied-expectations.md`](07-options-implied-expectations.md).
- **Raw prices are out of scope for the redistributable product (by design).** US equity
  prices are a commodity with no public-domain source; redistribution requires paid exchange/
  vendor licensing (hundreds–thousands/mo) for data that is *not* our differentiator. ptdata's
  value is point-in-time *corporate* data (restatements, survivorship-free membership, filing
  lineage, earnings timestamps) — the thing agents can't get elsewhere. For prices, use a
  dedicated source (free for personal use: e.g. your broker, or public EOD sources). We *do*
  serve **derived** event analytics computed internally — a realized earnings reaction (a
  return %) and the options-implied expected move (a %) — which are differentiated and are not
  redistribution of raw price/quote data. A price feed may be added later only if a customer
  demand and an affordable redistribution license both materialize.
- **No intraday data (MVP).** Option chains are end-of-day grain. Intraday event reactions are
  out of scope for the MVP.
- **Universe = all EDGAR for fundamentals & filings; index *membership* = S&P 500 only.**
  Filings and fundamentals are ingested for the full EDGAR universe (~7,992 CIKs from SEC's
  `company_tickers.json`). Survivorship-free *index membership* is tracked for the S&P 500 only;
  Russell 3000 / broad-index historical membership is a paid/hard data problem, out of MVP scope.

## Coverage (measured against the live database)

- **Fundamentals & earnings:** measured period floor is **FY2009** (the XBRL `companyfacts`
  era); earnings events (8-K Item 2.02) span the full filing history. Prices are out of scope.
- **EDGAR filings:** full history is ingested (both `filings.recent` and the older
  `filings.files[]` pages). Fundamentals depth is therefore bounded by XBRL availability
  (~2009), not by filing coverage. See [`02-edgar-ingestion.md`](02-edgar-ingestion.md).
- **Pre-XBRL fundamentals:** SEC XBRL `companyfacts` begins ~2009, so the *XBRL tier* floors at
  FY2009. Earlier periods are served by the separate **parsed tier** — values read from filing
  text for a *curated* set (currently Enron Q3 2001 and WorldCom Q1 2002, human-`verified`),
  clearly labeled `provenance='parsed'` and gap-fill only. This is curated, not comprehensive,
  and has its own extraction error profile; see [`09-pre-xbrl-fundamentals.md`](09-pre-xbrl-fundamentals.md).
- **Fundamentals (current):** only **duration** concepts (income-statement) are ingested;
  instant/balance-sheet concepts are a follow-on. Concept→tag coverage is partial (e.g.
  revenue spans `Revenues` and ASC-606 tags; not all are mapped yet), so some metrics are
  sparse. Facts whose source filing is outside the ingested window are skipped (lineage
  integrity), bounding fundamentals depth to filing coverage.
- **Upstream fiscal-year sanity.** A few filers report a bogus XBRL `fy` — Excel serial dates
  (e.g. 43101 = 2018-01-01), typos (1905, 2107), or null. ptdata does not anchor a period on an
  implausible fiscal year (accepted range `[1990, 2100]`); such facts are skipped rather than
  stored as junk years or aborting the company's ingest. (Initial backfill surfaced 46 such
  facts across the universe, since removed; one company that failed on a null `fy` was recovered.)
- **Index membership reconstruction:** S&P 500 membership is reconstructed from a free source
  (Wikipedia current + changes tables), treated as a source, not an authority. Where the
  change record is incomplete, the reconstruction errs toward **under-inclusion**: unbounded
  dangling additions are dropped, and overlapping per-ticker intervals are deduped (most
  recent kept). Historical member counts are therefore approximate (typically ~500–515).
  Securities are keyed by ticker pending CIK linkage. See [`05-index-membership.md`](05-index-membership.md).
- **Restatement vs. recast:** facts are attributed to their actual period (by end date), so
  prior-year comparative *columns* no longer contaminate the filing's own period. A later
  filing reporting a *different* value for a period still creates a new knowledge-time version
  — this captures genuine restatements but also comparative *recasts/reclassifications* (e.g.
  continuing-vs-total operations) the same way; the two are not distinguished as to cause.
  See [`03-fundamentals.md`](03-fundamentals.md).
- Options-implied expectations: bounded by the chosen historical-options vendor's depth.
  ORATS near-EOD reaches ~2007; OptionMetrics IvyDB reaches 1996 at institutional pricing.
  The actual depth at launch will be stated here once the vendor is fixed.

## Open items to confirm before launch

- **OPRA historical redistribution carve-out.** Our basis for redistributing options-derived
  analytics rests on the OPRA Vendor Agreement's historical-data provision plus the fact that
  derived analytics are not raw OPRA data. This is read from the primary agreement but **not
  yet confirmed with OPRA/counsel**, and the chosen vendor's own terms may add restrictions.
  Until confirmed, treat the options-implied feature's commercial redistribution as
  provisional.
- **Price-data redistribution (resolved → out of scope).** Research confirmed cheap vendors
  forbid redistribution under standard tiers and gate it behind enterprise/contact-sales deals
  (EODHD ~$2,499/mo; Twelve Data / FMP / Sharadar / Intrinio enterprise-only); only Polygon/
  Massive's Business ToS permits re-serving to a product's own end users, and EOD/delayed data
  is largely "not fee liable" under the SIP rules. Rather than spend on the non-differentiating
  commodity pre-revenue, raw prices are **out of scope by design** (see above); this can be
  revisited via Polygon/Massive if demand appears.
- **Dissemination timestamps.** `dissemination_ts` (true wire/press-release time) is
  best-effort and frequently `estimated`/`unknown` for older events; only `sec_acceptance_ts`
  is guaranteed exact. Event windows fall back to SEC acceptance when dissemination
  confidence is not `exact`, and the response says which anchor was used. See
  [`06-earnings-events.md`](06-earnings-events.md).

## Methodological caveats

- **Append-only with knowledge-interval close-out.** We close a superseded version's
  `knowledge_to` in place (a temporal bookkeeping write); we never alter a stored value. This
  is the standard pragmatic bitemporal model, documented in
  [`01-bitemporal-model.md`](01-bitemporal-model.md) §3.
- **Options-implied expected move is model-light but not assumption-free.** The MVP
  `atm_straddle` method is robust; the optional risk-neutral-density method is sensitive to
  strike-wing quality. Each estimate carries its `method` and `methodology_version`.
- **Restatement detection depends on filing linkage.** We link restatements via EDGAR
  amendment relationships and period-of-report matching. Restatements made through means we
  cannot link automatically may be missed until reconciled; such cases, once found, become
  golden test cases.

## What a query returns when we don't know

A point-in-time query whose knowledge coordinate predates our earliest knowledge of a fact
returns **nothing for that coordinate** — explicitly, not a back-filled present-day value.
Absence is a first-class, truthful answer.
