# 07 — Options-implied expectations

This document replaces what, in an earlier design, would have been an analyst-consensus
methodology. ptdata does not serve analyst consensus. This section explains **what we serve
instead, why, and on what legal and methodological basis.**

## Why not analyst consensus

The lookahead-bias problem has a famous instance in earnings: backtesting against the
analyst consensus *as it looks today* rather than the consensus that existed *before* the
announcement. Solving it properly requires genuine point-in-time consensus — dated snapshots
of what consensus was on each past date, never revised after the fact.

We investigated whether such data could be sourced cheaply and, critically, **redistributed**
through a commercial API. The conclusion was unambiguous:

- Affordable vendors (FMP, Finnhub, Intrinio, Benzinga) expose only the *current* consensus,
  or only ratings/price targets — not dated historical snapshots of EPS/revenue consensus.
- The two affordable sources of *genuine* point-in-time consensus history (Zacks; Estimize,
  the crowdsourced platform) are excellent but their licenses **forbid redistribution** —
  they are usable for private research, not for re-serving to customers.
- True point-in-time consensus as a redistributable product exists only at institutional
  pricing (LSEG/Refinitiv I/B/E/S Point-in-Time, Visible Alpha, FactSet).
- Building a consensus from individual analyst estimates is impractical: the per-analyst
  components are themselves proprietary and contractually gated.

Serving a cheap imitation — e.g. a "historical consensus" reconstructed by back-filling
today's estimates — would reintroduce exactly the lookahead bias this product exists to
eliminate, and would fail our own correctness tests. We declined to do it.

## What we serve: the options-implied expected move

Instead we serve the expectation the **options market** had priced in immediately before
each announcement. This is a *different object* from analyst consensus, and we never label
it "consensus":

- Analyst consensus answers *"what will EPS be?"* — a fundamental point estimate.
- The options-implied expected move answers *"how large a move is the market braced for?"* —
  a magnitude derived from option prices, direction-agnostic.

For an event-driven strategy this is arguably the more directly useful quantity: it is the
benchmark the position is actually traded against. And it has a decisive property —

> **It is point-in-time-correct by construction.** An option's closing price on a given date
> simply *is* what it was on that date. There is no "revision" to leak backward, no
> survivorship to launder. The expected move computed from a historical chain snapshot is
> inherently as-of that snapshot.

## Computation

- **Primary method (`atm_straddle`, MVP):** from the end-of-day chain for the expiration
  bracketing the event, take the at-the-money straddle (nearest strike to the underlying)
  and derive the implied 1-event expected move. Robust, transparent, easy to reproduce.
- **Optional method (`rnd_breeden_litzenberger`, post-MVP):** recover the full risk-neutral
  density from the strike curve (Breeden–Litzenberger), yielding implied skew and tail
  probabilities. More information, more sensitivity to wing quality; versioned separately.

Each `implied_expectation` row records its `method`, its `methodology_version`, and lineage
(`source_chain_ids`) to the exact `options_chain_eod` rows it was computed from. Changing the
computation is a new methodology version, not a silent rewrite.

## Data source and historical depth

The input is historical end-of-day options chains with implied volatility and greeks.
Candidate sources (evaluation ongoing; see `08`): ORATS near-EOD (depth to ~2007),
OptionMetrics IvyDB US (depth to 1996, institutional pricing), IVolatility, Cboe DataShop.
The chosen vendor and the resulting coverage window are recorded in
[`08-known-limitations.md`](08-known-limitations.md) at launch.

## Licensing basis for redistribution

Real-time OPRA options data is heavily licensed (a redistributor agreement and per-category
fees). **Historical** options data is treated differently. The OPRA Vendor Agreement
provides that historical OPRA data may be furnished to any person (data becomes "historical"
at the next session's open), and per-user display/redistributor fees apply to real-time/
delayed data, not historical. Moreover the derived analytics we serve (an expected-move
percentage, an implied volatility, a risk-neutral density) are not the raw OPRA quote data.
ptdata is an entirely historical product, so the expectations we serve sit on this footing.

> **Status: to be confirmed before launch.** This carve-out is read from the primary OPRA
> Vendor Agreement but has not yet been confirmed with OPRA/counsel, and the chosen vendor's
> *own* contract may impose additional terms (some vendors restrict redistribution of
> derived works on self-serve tiers). We will not represent the redistribution basis as
> settled until both are confirmed; tracked in [`08-known-limitations.md`](08-known-limitations.md).
