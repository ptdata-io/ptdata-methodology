# 00 — Overview

This document explains, in concrete terms, what ptdata claims to do and how the
architecture delivers it. It is written to be checked: every claim here corresponds to a
behavior exercised by the [ptdata-correctness](https://github.com/ptdata/ptdata-correctness)
test suite.

## The single claim

> Every value ptdata returns is the value that was **knowable on a date you specify**,
> carrying the lineage that proves where it came from and when we learned it.

Everything else is mechanism in service of that claim.

## Two time axes

A point-in-time data store has to separate two questions that single-timestamp databases
conflate:

- **Valid time** — *when was this fact true in the world?* Apple's revenue for fiscal 2017
  is "true" for the fiscal-2017 period regardless of when it was reported.
- **Knowledge time** — *when did we learn it?* The originally-reported FY2017 number became
  known to us on the 10-K filing date; a restated number became known later, on the
  amendment's filing date.

Storing both lets a query name a coordinate in the plane: *"as of knowledge date K, what
was the value for valid time V?"* A conventional database storing only "the current value"
can answer only the degenerate case K = now — which is lookahead bias by construction.

## Worked example 1 — a restatement (fundamentals)

General Electric's fiscal-2017 10-K reported certain figures that GE later restated in
2018. Two facts coexist in the store for the same valid time (FY2017):

| valid_time | value | knowledge_from | knowledge_to | source (accession) |
|---|---|---|---|---|
| FY2017 | *original* | 2018-02-23 (10-K filed) | 2018-11-?? | `0000040545-18-000010` |
| FY2017 | *restated* | 2018-11-?? (10-K/A) | ∞ | `0000040545-18-0000NN` |

- `get_fundamentals("GE", as_of_date="2018-02-28", ...)` returns the **original** figures —
  because on 2018-02-28 the restatement did not yet exist. (Knowledge coordinate falls in
  the first row's `[knowledge_from, knowledge_to)`.)
- `get_fundamentals("GE", as_of_date="2019-03-01", ...)` returns the **restated** figures,
  and the lineage shows the amendment chain linking the two accession numbers.

Nothing was overwritten. The original is still there, still queryable, forever. A backtest
that ran "in early 2018" sees exactly what it would have seen then.

## Worked example 2 — point-in-time price adjustment (corporate actions)

Apple executed a 7-for-1 split effective June 9, 2014. The raw daily prices are stored
once and **never mutated**. Split-adjusted prices are *computed on demand* from the
`corporate_action` records — using only the actions whose knowledge time is on or before
your `as_of_date`.

- A backtest dated **2014-05-01** asks for adjusted prices. The split had been *announced*
  (April 23, 2014) but a backtest configured to its announcement-knowledge sees the
  adjustment; a backtest dated **2014-04-01**, before any split was known, sees the
  unadjusted series. The adjusted price series is therefore itself a point-in-time object,
  not a fixed artifact of today's split history.
- `get_price_history` returns raw and adjusted alongside the adjustment factor and the
  corporate-action lineage that produced it, so the adjustment is auditable rather than
  opaque.

Most retail data APIs return *today's* fully-adjusted series for any historical query. That
is a lookahead leak whenever a split or spinoff occurred after the backtest date.

## Worked example 3 — the priced-in expectation (earnings events)

For NVIDIA's Q1 FY2024 announcement (May 24, 2023), ptdata records the 8-K acceptance
timestamp, the as-reported EPS and revenue (lineage to the 8-K accession), the
**options-implied expected move** as it stood at the prior close (T-1), and the realized
price reaction across a configurable window (default T-5…T+5). The "surprise" the product
exposes is the realized move measured against the move the options market had *priced in* —
not against an analyst consensus. See [`06-earnings-events.md`](06-earnings-events.md) for
why, and [`07-options-implied-expectations.md`](07-options-implied-expectations.md) for how
the expected move is computed and why it is point-in-time-correct by construction.

## Architecture in one paragraph

PostgreSQL 16 holds a bitemporal, append-only store (schema:
[`01-bitemporal-model.md`](01-bitemporal-model.md)). Ingestion jobs pull from SEC EDGAR
(filings, fundamentals, 8-K earnings timestamps — public domain), a daily-price source, a
reconstructed S&P 500 membership history, and a historical options-analytics source. The
same business logic is exposed through two surfaces: an **MCP server** (the primary,
agent-native interface) and a **REST API** under `/v1/` with equivalent semantics. Every
response embeds the source, knowledge timestamp, and methodology version of each value
returned.

## What this architecture refuses to do

- It will not overwrite a value when a newer one arrives. (Append-only; see commitment in
  `01`.)
- It will not return a fact without its provenance.
- It will not silently fill a coverage gap with a present-day value. A query whose
  knowledge coordinate predates our earliest knowledge of a fact returns *nothing for that
  coordinate*, explicitly — not a back-filled guess.
