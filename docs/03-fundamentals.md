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
knowledge-time version. Logical `revenue` keeps the headline concept first filed for that
period. A tag that appears only in a later filing does not replace it. Two headline tags filed
the same day break toward the more specific tag (`RevenueFromContractWithCustomer…` before
legacy `Revenues`). Later as-of dates follow restatements of the chosen concept.

A restatement falls straight out of the model: a later filing reporting a *different* value
for the same period creates a new knowledge-time version and closes the prior one — nothing is
overwritten (methodology §4). A point-in-time query returns the version knowable on the
requested date. The earnings-event `as_reported` revenue and `eps_gaap` are the values
knowable on the 8-K acceptance date. When the XBRL for the period arrives with the later
10-K, that is the first version of the headline concept, not a subsequent restatement and not
a tag adopted afterward.

## Logical metrics across us-gaap tags

Filers tag the "same" line item under different us-gaap concepts and change tags over time
(e.g. legacy `Revenues`, ASC-606 `RevenueFromContractWithCustomer…`, `SalesRevenueNet`; banks
and insurers differ again — they have no single revenue line at all, so they get an as-reported
*revenue family*, see [`10-financials.md`](10-financials.md)). We ingest **each tag as its own faithful concept** (no lossy
merging) and expose **logical metrics** (`revenue`, `net_income`) that resolve to one
underlying tag **for that fiscal period**. For `revenue` that tag is the one first filed for
the period; same-day ties follow candidate order, and a later tag does not take over. Other
logical metrics use the first populated candidate. For "latest", the
target year is the most recent across *all* candidate tags — so a discontinued tag
(e.g. a company that stopped using the ASC-606 tag in 2021) does not return a stale year.
Validated across tech, banks, energy, insurers, and retail mega-caps. This is validated against real data: e.g. GE's
2019-Q3 net income was reported one way originally and re-presented later; a query "as of"
the earlier date returns the original figure, "as of" the later date the revised one.

## Share counts

Share counts are served as three logical metrics, XBRL era:

- **`shares_outstanding`** — common shares outstanding **at the balance-sheet date** (an
  *instant* concept, like the balance-sheet lines), from `CommonStockSharesOutstanding`.
- **`weighted_avg_shares_diluted`** / **`weighted_avg_shares_basic`** — the *period-average*
  counts that denominate diluted / basic EPS, from
  `WeightedAverageNumberOfDilutedSharesOutstanding` /
  `WeightedAverageNumberOfSharesOutstandingBasic`.

They carry the unit `shares` (a pure count) and follow the same as-reported, point-in-time,
restatement-aware model as every other metric: a later filing that restates a share count
(e.g. a retroactive split adjustment) creates a new knowledge-time version, and a query "as of"
the earlier date returns the originally-reported count.

We **never sum share classes.** Each logical metric resolves to a single reported concept per
period; a dual-class filer that reports only per-class counts with no printed total yields no
fabricated total — the same as-reported discipline applied to the revenue family. Together with
prices (a follow-on), `shares_outstanding` is the primitive that makes point-in-time market
capitalization reconstructable without look-ahead bias.

Annual (FY) figures remain the filed coordinate. A derived Q4 is that annual figure minus the
three quarters of the same concept. It is labeled `provenance: "derived"` and is not a second
annual total. An individual quarterly value can occasionally reflect a filer's own scale-tagging
error in that 10-Q; later filings supersede it, and the annual figure is the filed total.
EPS and weighted-average share counts are not derived. Share counts before the XBRL era are a
documented follow-on. Derived quarters and `cost_of_revenue` are XBRL-era only.

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

## Logical metric names

`LOGICAL_METRICS` has 25 names. Twelve are the original statement lines (revenue, operating
income, EBIT, net income, diluted EPS, operating cash flow, capex, total assets, total
liabilities, equity, cash, long-term debt). The map also has `gross_profit`, the revenue
family (`total_revenue`, `total_net_revenue`, `net_interest_income`, `noninterest_income`,
`premiums_earned`, `net_investment_income`, `real_estate_revenue`), three share counts,
`cost_of_revenue`, and `eps_basic`.

`cost_of_revenue` is sticky, like `revenue`. The candidate order is `CostOfRevenue`,
`CostOfGoodsAndServicesSold`, `CostOfGoodsSold`, `CostOfServices`. An empty higher-priority
tag does not block a populated later tag. Tags whose name says excluding depreciation are
not candidates. If the only populated cost tags are the product line and the service line,
`cost_of_revenue` is unavailable. Those two lines are not added. `gross_profit` is the
tagged `GrossProfit` fact. It is not computed as revenue minus cost. `ebit` has no us-gaap
tag in this map. It stays `unavailable` with `code: "not_reported"` unless a parsed fact
was stored under that name. `eps_basic` is `EarningsPerShareBasic` and follows the same
non-derived Q4 rule as diluted EPS.

## Derived quarters (XBRL era)

The read path derives a quarter only when the stored fact for that period is missing.
A value served from `financial_fact` is unchanged and has no `derivation` object.

A quarter-length span (about 76–105 days) that ends on the fiscal year-end is stored as
that year's Q4. A later 10-Q often stamps its own fiscal period on that comparative
column. The dates still make it Q4, so the read path serves the tagged amount and does
not subtract. Target's FY2025 net income is the case: the tagged quarter is
1,046,000,000 and the annual figure minus the three quarters is 1,045,000,000.

Flow Q4, including `cost_of_revenue`, is `FY − (Q1 + Q2 + Q3)` on one us-gaap concept.
The method is `fy_minus_q1_q3`. A same-concept nine-month total, when one is stored, must
agree within 1 currency unit or the quarter is withheld. `provenance` is `"derived"`.
`source_accession` is the 10-K accession of the FY input. `knowledge_from` is the latest
filing date among the inputs. `methodology_version` stays `0.1.0`.

The response adds `derivation.inputs` (period, accession, knowledge time, value) and copies
those accessions onto `lineage.derived_inputs`. Both keys are omitted when nothing was
derived. `extraction_confidence` stays present and null on XBRL and derived values.

```json
{
  "value": 94930000000,
  "unit": "USD",
  "fiscal_year": 2024,
  "fiscal_period": "Q4",
  "source_accession": "0000320193-24-000123",
  "knowledge_from": "2024-11-01T00:00:00Z",
  "methodology_version": "0.1.0",
  "provenance": "derived",
  "extraction_confidence": null,
  "derivation": {
    "method": "fy_minus_q1_q3",
    "inputs": [
      {
        "fiscal_period": "FY",
        "accession": "0000320193-24-000123",
        "knowledge_from": "2024-11-01T00:00:00Z",
        "value": 391035000000
      }
    ]
  }
}
```

An `as_of` date before the FY 10-K was filed does not see that Q4. Inputs are the versions
whose filing date is on or before `as_of`. A later amendment creates a new derived version.
An earlier `as_of` still sees the previous one.

The quarter is withheld, with `code` still `not_reported`, when any of these fail:

- A required input is missing. The reason names the slot.
- Durations disagree with a year: FY about 350–380 days, each quarter about 76–105 days,
  Q1 starting on the FY start, quarters abutting, no gap and no overlap. A 10-KT, a
  transition report, or a short year is not derived.
- The four inputs mix a first-filed figure with a comparative restatement from a later
  ordinary 10-Q or 10-K. Microsoft FY2017 revenue is this case and is not served.
  A `10-Q/A` or `10-K/A` updates that one input and moves `knowledge_from` without waiting
  for the other three.
- Revenue, the other revenue totals, `cost_of_revenue`, or capex would come out negative.

Operating cash flow and capex for Q2 and Q3 keep a filed quarter of about three months.
Microsoft FY2023 Q2 operating cash flow stays the discrete fact. If that quarter is absent,
Q2 is YTD6 minus Q1 and Q3 is YTD9 minus YTD6 (`ytd_minus_prior_ytd`). Those differences
are knowable when the 10-Q that carries the cumulative figure is filed. They do not wait
for the 10-K. Q4 still waits for the FY fact. The source accession is the later
year-to-date filing. Capex stays positive for cash paid.

Instant balance-sheet lines and shares outstanding at Q4 copy the FY instant. Provenance
stays the filing's provenance. The method is `fy_instant`.

EPS and weighted-average share counts are not additive. A filed quarter-length Q4 fact is
served as filed. Otherwise the reason is exactly `Q4 EPS/share count is not additive; not derived`.

Forms that may feed a derivation are `10-Q`, `10-Q/A`, `10-K`, and `10-K/A`. 8-K facts and
the parsed pre-XBRL tier are ignored by the derivation. Foreign 20-F and 40-F filers are
unchanged.

## To document here

Concept normalization and unit handling; how partial restatements of a single line item are
represented when they are not an amendment; treatment of as-reported figures that first
appear in an 8-K exhibit versus the subsequent 10-Q.
