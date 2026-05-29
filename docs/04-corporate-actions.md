# 04 — Corporate actions & point-in-time price adjustment

*Scaffold — to be completed alongside price ingestion (build step 6d).*

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
