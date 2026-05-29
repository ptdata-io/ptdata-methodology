# 06 — Earnings events

An earnings event in ptdata is the join of four things, each point-in-time-correct:

1. **When the announcement happened** — timestamp(s).
2. **What was reported** — as-reported figures, with lineage.
3. **What was priced in beforehand** — the options-implied expected move at T-1
   (see [`07-options-implied-expectations.md`](07-options-implied-expectations.md)).
4. **What actually happened** — realized daily price action across a configurable window
   (default T-5…T+5).

`get_earnings_event(ticker, fiscal_period, window=…)` returns all four.

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

A T+1 reaction measured from `sec_acceptance_ts` when the news actually broke earlier (or
later) is mis-anchored. ptdata therefore stores both. `dissemination_ts` is nullable and
carries a `dissemination_confidence` flag (`exact` | `estimated` | `unknown`) and a
`dissemination_source`. The event window anchors on `dissemination_ts` when its confidence
is `exact`, otherwise it falls back to `sec_acceptance_ts` and says so in the response.

> **Honest limitation.** Sourcing exact wire-dissemination times historically is hard.
> Where we cannot establish one with confidence, we expose the SEC acceptance timestamp and
> flag the dissemination time as `estimated`/`unknown` rather than guessing. See
> [`08-known-limitations.md`](08-known-limitations.md).

## As-reported figures

EPS and revenue (and other reported headline figures) are stored as `financial_fact` rows
whose `source_accession` is the **8-K** that announced them — *not* the later 10-Q/10-K. The
8-K Item 2.02 press-release exhibit is what the market reacted to; the 10-Q filed weeks
later may differ slightly (and any such difference is itself visible as a later-knowledge
version of the same fact). The earnings event therefore reports the figures *as announced*,
with lineage to the announcing 8-K.

We expose as-reported values; we do **not** compute a beat/miss against an analyst
consensus, because ptdata does not carry analyst consensus (see
[`07`](07-options-implied-expectations.md) for the rationale). The expectation benchmark we
*do* provide is the options-implied expected move.

## Event windows

The price-action window is configurable (`window=(-5, +5)` default), measured in trading
days relative to the event day T. T is the trading session in which the announcement was
disseminated (after-close announcements set T = the session just ended; before-open
announcements set T = the upcoming session). The window returns daily raw and adjusted
prices and the per-day return, point-in-time-adjusted per
[`04-corporate-actions.md`](04-corporate-actions.md).

The headline derived statistic is the **realized move vs. implied move**: the actual
T-1→T+1 (configurable) return compared against the options-implied expected move that stood
at T-1. For NVIDIA Q1 FY2024 (announced 2023-05-24), a roughly +24% realized reaction
against a far smaller implied move is the kind of "surprise relative to what was priced in"
this feature is built to quantify.

## Scheduling / upcoming events

`list_upcoming_earnings(date_range, universe)` returns scheduled events with a confirmation
status. Note this is the one forward-looking surface in an otherwise historical product; it
depends on a calendar source and is best-effort as to confirmation. It does not participate
in point-in-time guarantees (there is no "knowledge time" for an event that hasn't happened).
