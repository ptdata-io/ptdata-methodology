# 08 — Known limitations

The moat of this product is honesty about what it does and does not guarantee. This document
is the canonical statement of current limitations and coverage gaps. It is updated whenever
behavior changes. If you find a gap not listed here, that is a bug in this document — please
open an issue on [ptdata-correctness](https://github.com/ptdata-io/ptdata-correctness).

## Status

**Live public API:** [https://api.ptdata.io](https://api.ptdata.io). Full-universe EDGAR backfill
complete; pre-XBRL parsed tier in production and growing; daily EDGAR incremental ingestion keeps
fundamentals/filings fresh. The figures below are *measured against the production database*
(2026-07-23), not targets. Coverage is gated by the public correctness suite (currently
**`12 passed, 6 skipped, 0 failed`** against `https://api.ptdata.io`; skipped cases mark
out-of-scope behavior, never a silent pass).

| Measure | Value |
|---|---|
| Companies (EDGAR universe + curated historical) | ~8,318 |
| — with fundamentals (XBRL or parsed tier) | ~6,306 (~76%) |
| — filings-only (no fundamentals — funds/trusts/foreign/small filers) | ~2,012 (~24%) |
| Filings ingested (all form types, full history) | ~7.25M |
| Earnings events (8-K Item 2.02) | ~237,400 |
| Financial facts (point-in-time) | ~2.53M |
| Fundamentals coverage | FY2009+ (XBRL), plus a growing parsed pre-XBRL tier |
| S&P 500 membership (distinct securities, historical) | ~1,256 |
| DJIA / NDX / DJT / DJU membership | served (depth varies — see below) |

`LOGICAL_METRICS` exposes **25** names. The original statement set is 12: revenue, operating
income, EBIT, net income, diluted EPS, operating cash flow, capex, total assets, total
liabilities, equity, cash, and long-term debt. The same map also includes `gross_profit`,
seven revenue-family names, three share counts, `cost_of_revenue`, and `eps_basic`. Older
copies of this page said 12 and meant that statement set. A listing that said 15 was that set
plus the three share counts. Banks, brokers, and insurers additionally get an as-reported
**revenue family** (they have no single revenue line); see [`10-financials.md`](10-financials.md).
Fundamentals are kept fresh by a daily
incremental ingestion of EDGAR's daily index (re-ingesting only CIKs that filed since the last run).

**Pre-XBRL parsed tier (`provenance='parsed'`).** Below the FY2009 XBRL floor (and for some
defunct issuers), a *curated + batch* clearly-labeled, lower-confidence tier is parsed from filing
text via a fault-tolerant multi-agent pipeline (2 independent extract agents → adversarial verify →
gate that auto-loads only on agreement + verify, else withholds). **Measured (2026-07-23):**
**~1,539 companies** with current parsed facts, **~103,781** current parsed facts
(~1,681 `verified`, ~99,376 `high`, ~2,724 `medium`). Still a growing subset of the pre-XBRL
universe, not comprehensive. Gap-fill only (XBRL always wins); never replaces structured data.
Values are as-reported by CIK (not normalized across entity-identity or accounting-basis changes —
see `09`). Full methodology + limitations:
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
  dedicated source (free for personal use: e.g. your broker, or public EOD sources). ptdata
  does not serve prices. The options-implied expected move is a
  derived percentage, specified but not yet live, and is not a redistribution of raw quotes.
  A price feed may be added later only if a customer
  demand and an affordable redistribution license both materialize.
- **No intraday data (MVP).** Option chains are end-of-day grain. Intraday prices are
  out of scope for the MVP.
- **Universe = all EDGAR for fundamentals & filings; index *membership* = SP500 + DJIA + NDX + DJT + DJU.**
  Filings and fundamentals are ingested for the full EDGAR universe (SEC `company_tickers.json` plus
  curated historical names). Survivorship-free *index membership* is served for those five indices
  with **index-specific depth** (SP500/DJIA deepest; NDX monthly modern-only; DJT/DJU modern floors).
  Russell 3000 / multi-decade broad membership is a paid/hard data problem, out of MVP scope.
- **Options-implied expected move is not yet live in the public API.** The methodology is
  documented in [`07`](07-options-implied-expectations.md); product serving awaits licensed
  options history. Correctness cases that need it are **skipped**, not green-washed.

## Derived quarters (XBRL era)

A missing Q4 on a flow line is served only as the annual total minus the three discrete
quarters of the same us-gaap concept, and only when the durations abut, the forms are 10-Q,
10-Q/A, 10-K, or 10-K/A, and the four inputs are not a mix of a first-filed figure and a
comparative restatement. An amendment updates that one input. A later ordinary 10-Q that
restates a quarter while the annual total is still the first filing does not produce a Q4.
Microsoft FY2017 revenue is the worked case. EPS and weighted-average share counts are not
subtracted. Balance-sheet Q4 is the FY instant and keeps the filing's provenance. Operating
cash flow and capex keep a filed quarter of about three months, and otherwise use the
year-to-date difference. `cost_of_revenue` follows the same flow rules. It is not a sum of
product cost and service cost. `gross_profit` is not computed from revenue and cost. `ebit`
stays unavailable unless a fact was stored under that name. The parsed pre-XBRL tier does
not derive Q4 and does not extract `cost_of_revenue`. Derived values are computed on read
and are not written into `financial_fact`.

## Coverage (measured against the live database)

- **Fundamentals & earnings:** measured period floor is **FY2009** (the XBRL `companyfacts`
  era); earnings events (8-K Item 2.02) span the full filing history. Prices are out of scope.
- **EDGAR filings:** full history is ingested (both `filings.recent` and the older
  `filings.files[]` pages). Fundamentals depth is therefore bounded by XBRL availability
  (~2009), not by filing coverage. See [`02-edgar-ingestion.md`](02-edgar-ingestion.md).
- **Pre-XBRL fundamentals:** SEC XBRL `companyfacts` begins ~2009, so the *XBRL tier* floors at
  FY2009. Earlier periods are served by the separate **parsed tier** — values read from filing
  text for a growing set (~1.5k companies / ~104k facts as of 2026-07-23, expanding via
  prioritized batches), clearly labeled `provenance='parsed'` and gap-fill only. This is partial,
  not comprehensive, has its own extraction error profile, and is as-reported by CIK (not
  normalized across entity-identity or accounting-basis changes); see
  [`09-pre-xbrl-fundamentals.md`](09-pre-xbrl-fundamentals.md).
- **Fundamentals (current):** full statements are ingested — income statement, **balance sheet,
  and cash flow** (25 logical metric names; the original statement set is 12; instant
  balance-sheet concepts are stored as point-in-time snapshots). Concept→tag coverage is
  resolved across multiple us-gaap tags per metric (e.g.
  revenue spans `Revenues`, ASC-606, and `SalesRevenueNet`; long-term debt and capex coalesce
  several variant tags), so a metric a filer doesn't tag is sparse/absent rather than wrong. Facts
  whose source filing is outside the ingested window are skipped (lineage integrity), bounding
  fundamentals depth to filing coverage.
- **Financials (banks / brokers / insurers).** These filers have no single "revenue" line, so
  rather than synthesize one we serve an as-reported **revenue family** (`net_interest_income`,
  `noninterest_income`, `total_net_revenue`, `premiums_earned`, `net_investment_income`, and
  `total_revenue` where a grand total is printed); a financial is covered when it reports ≥1
  member. Logical `revenue` returns a printed headline when the filer has one, including
  `RevenuesNetOfInterestExpense`. The headline is the concept first filed for that period, and
  later as-of dates follow that concept's restatements. It does not sum components. An unavailable `revenue` (no headline)
  returns guidance with a reason code pointing to the family rather than a null. Residual uncovered
  cases are structural — revenue-less ETF/crypto trusts (SIC 6221) and
  foreign IFRS-tagged filers. Full methodology: [`10-financials.md`](10-financials.md).
- **Upstream fiscal-year sanity.** A few filers report a bogus XBRL `fy` — Excel serial dates
  (e.g. 43101 = 2018-01-01), typos (1905, 2107), or null. ptdata does not anchor a period on an
  implausible fiscal year (accepted range `[1990, 2100]`); such facts are skipped rather than
  stored as junk years or aborting the company's ingest. (Initial backfill surfaced 46 such
  facts across the universe, since removed; one company that failed on a null `fy` was recovered.)
- **Index membership reconstruction:** S&P 500 membership is reconstructed from a dated historical
  constituent roster (the community-maintained `fja05680` dataset, 1996+), treated as a source, not
  an authority. Every ticker→CIK mapping is **EDGAR-verified** (the ticker must appear on the
  issuer's 10-K cover in that era) or accepted on exact EDGAR-name identity — **no bare-ticker
  guesses**. Where the change record or the cover-page index is incomplete the reconstruction errs
  toward **under-inclusion**: unverified/dangling spans are omitted, never guessed, so coverage is a
  conservative floor that thins pre-2010 (as-of counts ≈ 504 in 2024, ≈ 490 in 2008, ≈ 448 in 2000).
  Each constituent carries its stable `cik` + current `name` alongside the era-correct as-of-date
  ticker. See [`05-index-membership.md`](05-index-membership.md).
- **DJIA membership reconstruction:** DJIA (`index` code `DJIA`) is reconstructed from the Wikipedia
  historical-components change chronology plus recent yfiua snapshot confirmation; stored source
  token **`wikipedia-djia+efts`**. Same interval reconstruction and era-correct ticker rules as
  SP500. **Policy B:** `cik` may be null for pre-EDGAR / unresolved identity; CIKs are never
  invented. Not an official S&P DJI feed. **Measured as-of counts (2026-07-12):** 2000-06-30 ≈ 29,
  2008-09-15 = 30, 2024-06-30 = 30 (modern decades near-complete; do not claim exactly 30 on every
  historical date). Golden cases: AAPL inclusion effective 2015-03-19; GE removal effective
  2018-06-26.
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
  is guaranteed exact. See
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
