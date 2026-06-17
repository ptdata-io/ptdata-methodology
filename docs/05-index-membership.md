# 05 — Index membership (survivorship-free, as-of-date)

*Status: implemented and served for the S&P 500. Reconstructed from a dated historical
constituent roster, with every ticker→CIK mapping EDGAR-verified.*

## How it works

We take a **dated historical constituent roster** (the community-maintained `fja05680` S&P 500
dataset: a current-members table plus a changes table of additions/removals back to 1996) and
**reconstruct** membership intervals by walking the change events from the current snapshot: an
"added on D" begins a membership at D; a "removed on D" ends one at D; a current member with no
relevant change has an unknown (pre-coverage) start and an open end. Intervals are stored
bitemporally in `index_membership` (valid time = membership span; knowledge time = our
reconstruction — see [`01-bitemporal-model.md`](01-bitemporal-model.md)). A point-in-time query
`get_index_constituents(index, as_of_date)` returns the constituents as understood on that date —
survivorship-free.

The roster is treated as a **source, not an authority**. Reconstruction quality is bounded by the
completeness of the change record; where it is incomplete we err toward **under-inclusion** (see
coverage floor below).

## Verified identity (ticker → CIK)

A ticker is not a stable identifier (symbols are reused and renamed), so a roster keyed only by
ticker cannot be reliably joined to point-in-time fundamentals. We therefore **verify every
ticker→CIK mapping** rather than guessing it: a mapping is accepted only when the ticker appears as
the trading symbol on that issuer's 10-K cover in that era (EDGAR full-text / cover-page check), or
on exact EDGAR-name identity. **No bare-ticker guesses are stored.** The ~hundreds of companies that
no longer exist (acquired, failed, taken private) are ingested specifically so the historical index
is complete, not just its survivors.

Each constituent returned therefore carries its **stable identity** alongside the as-of-date ticker:

- **`cik`** — the SEC Central Index Key, the stable join key into fundamentals/filings. (Null only
  for the few roster entries known by ticker but not yet CIK-linked — preserved, never guessed.)
- **`ticker`** — **era-correct**: a member that was renamed returns the symbol it actually traded
  under on `as_of_date` (e.g. AmerisourceBergen → `ABC` in 2008, `COR` from 2023; Facebook → `FB`
  pre-2022, `META` after). This makes a constituent list directly joinable to point-in-time data
  without lookahead.
- **`name`** — the company's **current** registered name (we persist one EDGAR name, not per-date
  historical names). So a 2008 query returns ticker `ABC` with name "Cencora, Inc." — the modern
  identity of that CIK. Join on `cik`; treat `name` as a current-identity label, not an as-of name.

## What this guarantees

`get_index_constituents("SP500", as_of_date)` returns membership as it was *understood on*
`as_of_date`, surviving the two classic biases:

- **survivorship** — dropped names (failed/acquired) are still present for the dates they were
  members, so a backtest is tested against the losers, not just today's survivors;
- **announce-ahead** — a name is added on its *effective* date, not its announcement date, unless
  queried by announcement knowledge time.

## Coverage floor (honest, conservative)

Because a ticker-span is included only when verified, coverage is a **conservative floor that
thins the further back you go**. Approximate as-of-date constituent counts: 2024 ≈ 504, 2015 ≈ 495,
2008 ≈ 490, 2005 ≈ 487, 2000 ≈ 448 (near-complete from ~2005 on). Unresolved spans — predominantly
pre-2001 (EDGAR full-text can't index them, so cover-page verification can't run) plus a handful of
identity ambiguities — are **omitted, never guessed**. A queried pre-2010 date may therefore return
slightly fewer than 500 names; every name returned is verified-correct, but the set is not
guaranteed complete for older dates. See [`08-known-limitations.md`](08-known-limitations.md).

## Worked reference (golden test cases)

- **Lehman Brothers (LEH)**: a member on 2008-09-14; absent by 2008-09-22 (removal effective
  2008-09-16). The membership row's `valid_to` is the removal effective date.
- **Tesla (TSLA)**: absent on 2020-12-20, present on 2020-12-21 (inclusion effective date).

## Scope

Deep survivorship-free **as-of-date membership is served for the S&P 500**. DJIA and Nasdaq-100
company *universes* have been added to fundamentals/filings coverage, with as-of-date membership
serving rolling out (current-forward for Nasdaq-100, where a reliable free deep history exists).
Russell 3000 / broad-index historical membership remains a paid/hard data problem, out of scope.
