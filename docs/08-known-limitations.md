# 08 — Known limitations

The moat of this product is honesty about what it does and does not guarantee. This document
is the canonical statement of current limitations and coverage gaps. It is updated whenever
behavior changes. If you find a gap not listed here, that is a bug in this document — please
open an issue on [ptdata-correctness](https://github.com/ptdata/ptdata-correctness).

## Status

**Pre-release.** Schema, methodology, and golden-file test cases exist; ingestion is in
progress. No coverage claim below should be relied on until the corresponding test cases
pass against the live service.

## By design (not gaps — deliberate scope decisions)

- **No analyst consensus.** ptdata does not carry analyst EPS/revenue consensus, current or
  historical. Genuine point-in-time consensus is not redistributable at an accessible price;
  a cheap reconstruction would reintroduce lookahead bias. We serve the **options-implied
  expected move** instead, which is a different quantity and is never labeled "consensus."
  Rationale: [`07-options-implied-expectations.md`](07-options-implied-expectations.md).
- **No intraday data (MVP).** Prices and option chains are end-of-day grain. Intraday event
  reactions are out of scope for the MVP.
- **Universe = S&P 500.** With historical membership tracked. Names outside the index are
  out of scope for the MVP.

## Coverage windows (targets; confirmed at launch)

- Prices: target 2000–present. Fundamentals & earnings: target 2010–present.
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
- **Price-data vendor & its redistribution terms.** The daily-price source is not yet fixed.
  Tiingo's terms forbid redistribution; Polygon/EODHD terms require direct confirmation. The
  fundamentals path does not depend on this (EDGAR is public domain), but the price surface
  does.
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
