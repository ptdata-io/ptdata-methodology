# 03 — Fundamentals: as-reported vs. restated

*Status: implemented (6c) from SEC EDGAR XBRL `companyfacts`, for duration (income-statement)
concepts. Instant/balance-sheet concepts are a documented follow-on.*

## How it works (6c)

Fundamentals come from EDGAR's XBRL `companyfacts` feed
(`https://data.sec.gov/api/xbrl/companyfacts/CIK##########.json`). Each reported entry
carries the source accession (`accn`), the period (`start`/`end`), the fiscal year/period
(`fy`/`fp`), and the date it was filed (`filed`). We map these to the bitemporal model:
`filed` → `knowledge_from`, the period → valid time, `fy`/`fp` → the logical fact identity.

**Fiscal-period attribution is by the fact's *actual* period (its end date), not by
companyfacts' `fy`/`fp`** — those reflect the *filing's* context, so a prior-year comparative
column would otherwise be mislabeled with the filing's period. We build an
end-date → (fiscal year, period) map from *primary* facts (where a filing's fact-end equals
its `period_of_report`) and attribute every fact by its end date. This places comparatives in
their true period and still captures a restated prior-year comparative as a new
knowledge-time version. Revenue is read from the ASC-606 tag with the legacy `Revenues` tag as
fallback.

A restatement falls straight out of the model: a later filing reporting a *different* value
for the same period creates a new knowledge-time version and closes the prior one — nothing is
overwritten (methodology §4). A point-in-time query returns the version knowable on the
requested date. The earnings-event `as_reported` figures are the originally-reported values
for the period (earliest knowledge time).

## Logical metrics across us-gaap tags

Filers tag the "same" line item under different us-gaap concepts and change tags over time
(e.g. legacy `Revenues`, ASC-606 `RevenueFromContractWithCustomer…`, `SalesRevenueNet`; banks
and insurers differ again). We ingest **each tag as its own faithful concept** (no lossy
merging) and expose **logical metrics** (`revenue`, `net_income`) that resolve to whichever
underlying tag is populated **for that fiscal period**, in a priority order. For "latest", the
target year is the most recent across *all* candidate tags — so a discontinued higher-priority
tag (e.g. a company that stopped using the ASC-606 tag in 2021) does not return a stale year.
Validated across tech, banks, energy, insurers, and retail mega-caps. This is validated against real data: e.g. GE's
2019-Q3 net income was reported one way originally and re-presented later; a query "as of"
the earlier date returns the original figure, "as of" the later date the revised one.

## Principle

ptdata stores fundamentals **as reported at each point in knowledge time**. The
originally-reported value and every later restatement coexist as bitemporal versions of the
same logical fact (same `company_id`, `concept_id`, fiscal period; differing knowledge
intervals). See [`01-bitemporal-model.md`](01-bitemporal-model.md).

## What this guarantees

- `get_fundamentals(ticker, as_of_date, metrics)` returns the values that were knowable on
  `as_of_date` — original figures before a restatement, restated figures after — never a
  blend, never silently the latest.
- The lineage in the response names the `source_accession` and, for restated values, the
  amendment chain.

## Worked references (become golden test cases)

- **GE FY2017**: original 10-K figures as-of 2018-02-28; restated figures as-of 2019-03-01.
- **Enron Q3 2001**: originally-reported figures as-of 2001-11-15 (pre-bankruptcy), not
  later forensic restatements.

## To document here

Concept normalization and unit handling; fiscal-period boundary rules; how partial
restatements (single line items) are represented; treatment of as-reported figures that
first appear in an 8-K exhibit vs. the subsequent 10-Q.
