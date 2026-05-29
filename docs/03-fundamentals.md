# 03 — Fundamentals: as-reported vs. restated

*Scaffold — to be completed alongside fundamentals ingestion (build step 6c).*

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
