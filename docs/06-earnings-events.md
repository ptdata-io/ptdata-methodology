# 06 — Earnings events

An earnings event in ptdata joins two things, each point-in-time-correct:

1. **When the announcement happened** — timestamp(s).
2. **What was reported** — as-reported figures, with lineage.

The options-implied expected move is specified in
[`07-options-implied-expectations.md`](07-options-implied-expectations.md) and is not yet
served. ptdata does not serve a realized price reaction.

`get_earnings_event(ticker, fiscal_year, fiscal_period)` returns the 8-K event.

*Status: implemented — 8-K Item 2.02 timestamps (6b), fiscal-period attribution, and
as-reported revenue and `eps_gaap`; served via `/v1/earnings-event`. The options-implied
expected move (6f) is omitted until that data lands. **Fiscal attribution:** an earnings 8-K is attributed to the fiscal period
of the periodic filing it precedes — a 10-Q gives the quarter, a 10-K gives FY. A Q4/annual
release accompanies the 10-K and is therefore attributed to **FY**, not Q4.*

## Timestamps: two distinct events, not one

This is the subtlety most data products get wrong. There are **two different moments**, and
conflating them corrupts any event study:

- **`sec_acceptance_ts`** — when EDGAR accepted the 8-K (Item 2.02, "Results of Operations
  and Financial Condition"). Recorded to the second in Eastern Time. Exact, free, and
  authoritative *for the filing event*.
- **`dissemination_ts`** — when the news actually reached the market: the press-release
  wire (Business Wire / GlobeNewswire / PR Newswire) or the company IR page. This is the
  market-moving event, and it frequently **precedes** the 8-K acceptance by minutes — and
  occasionally the 8-K is filed the next morning.

ptdata stores both. `dissemination_ts` is nullable and carries a `dissemination_confidence`
flag (`exact` | `estimated` | `unknown`) and a `dissemination_source`.

> **Honest limitation.** Sourcing exact wire-dissemination times historically is hard.
> Where we cannot establish one with confidence, we expose the SEC acceptance timestamp and
> flag the dissemination time as `estimated`/`unknown` rather than guessing. See
> [`08-known-limitations.md`](08-known-limitations.md).

## As-reported figures

`as_reported.revenue` and `as_reported.eps_gaap` are the values knowable when the 8-K was
accepted. The XBRL facts for the period usually arrive with the 10-K, after the 8-K, so
nothing is knowable on the acceptance date. The event then uses the first version of the
headline concept — the original 10-K — and not a later restatement or a tag the filer adopted
in a subsequent filing. `eps_gaap` is diluted EPS, or continuing-operations diluted EPS when
that is the only diluted figure. `eps_non_gaap` is not computed. The event's
`filing_accession` remains the announcing 8-K. The fact's own `source_accession` is the filing
that carried that version, usually the 10-K.

We expose as-reported values; we do **not** compute a beat/miss against an analyst
consensus, because ptdata does not carry analyst consensus (see
[`07`](07-options-implied-expectations.md) for the rationale).

## Scheduling / upcoming events

`list_upcoming_earnings(date_range, universe)` returns scheduled events with a confirmation
status. Note this is the one forward-looking surface in an otherwise historical product; it
depends on a calendar source and is best-effort as to confirmation. It does not participate
in point-in-time guarantees (there is no "knowledge time" for an event that hasn't happened).
