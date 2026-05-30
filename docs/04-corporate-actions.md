# 04 — Corporate actions & point-in-time price adjustment

*Status: adjustment methodology and split derivation implemented (6d); served via
`/v1/prices`. Note on data source below.*

## Data source (important)

Raw daily prices are, for now, ingested from a **broker feed (IBKR) for private research
only** — broker/exchange market data is licensed to the subscriber and is **not
redistributable** through the commercial product. So `/v1/prices` is populated on a private
instance, not the public redistributable surface, until a redistribution-licensed price
source is in place. The *methodology* below is source-independent.

## Split derivation (no vendor corporate-actions feed)

Because broker feeds don't cleanly expose split *events*, splits are **derived from the raw
price series**: a forward split appears as a clean, near-integer overnight change in price
level with no offsetting news (`prev_close / next_open ≈ N`, N ≥ 2). Transparent and
falsifiable. Limitations: reverse splits (ratio < 1) and dividends are not derived this way;
a rare ~N× single-session move could misfire. Documented in
[`08-known-limitations.md`](08-known-limitations.md).

## Point-in-time adjustment

A derived split's knowledge time is its **ex-date** (when prices actually adjust). The
adjusted series is computed on demand: for a bar on date *d*, the cumulative factor is the
product of `split_ratio` for splits with `ex_date > d` **and** knowable by the query's
`as_of_date`. Consequences: raw prices are never mutated; a split is applied only if it was
knowable by `as_of_date` (no lookahead); bars after `as_of_date` are excluded.

## Principle

Raw daily prices (`price_raw`) are stored once and **never mutated** by corporate actions.
Adjusted prices are *computed on demand* from `corporate_action` records, using only the
actions whose `knowledge_from <= as_of_date`. The adjusted series is therefore itself a
point-in-time object. See [`01-bitemporal-model.md`](01-bitemporal-model.md) §7.

## What this guarantees

- `get_price_history(ticker, start, end, adjustment_mode, as_of_date)` returns raw price,
  adjusted price, the cumulative adjustment factor, and the corporate-action lineage that
  produced it.
- A split/spinoff that occurred *after* the backtest's `as_of_date` does not retroactively
  adjust earlier prices — eliminating a lookahead leak common to retail price APIs.

## Adjustment methodology (to make precise here)

- Split adjustment: cumulative product of split ratios with `ex_date > price date`.
- Dividend adjustment modes (total-return vs. price-only) and their factor formulas.
- Spinoff handling and linkage to the child `security` via `related_security_id`.

## Worked reference (golden test case)

- **AAPL 7-for-1 split, ex-date 2014-06-09**: unadjusted and adjusted prices across the
  split both returned correctly, with adjustment metadata.
