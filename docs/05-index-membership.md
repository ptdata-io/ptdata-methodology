# 05 — Index membership (survivorship-free, as-of-date)

*Status: implemented and served for **S&P 500, DJIA, NDX, DJT, and DJU**. Each index is
reconstructed from a dated historical change log / roster (depth varies by index — see
sections below), with ticker→CIK mappings EDGAR-verified where possible (Policy B for
unresolved identity — never invent CIKs).*

## How it works

We take a **dated historical constituent roster** (per index — see sections below) and
**reconstruct** membership intervals by walking the change events from the current snapshot: an
"added on D" begins a membership at D; a "removed on D" ends one at D; a current member with no
relevant change has an unknown (pre-coverage) start and an open end. Intervals are stored
bitemporally in `index_membership` (valid time = membership span; knowledge time = our
reconstruction — see [`01-bitemporal-model.md`](01-bitemporal-model.md)). A point-in-time query
`get_index_constituents(index, as_of_date)` returns the constituents as understood on that date —
survivorship-free. The query path is **index-generic** (`SP500`, `DJIA`, …); only the loaded
source data differs.

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

`get_index_constituents(index, as_of_date)` returns membership as it was *understood on*
`as_of_date`, surviving the two classic biases:

- **survivorship** — dropped names (failed/acquired) are still present for the dates they were
  members, so a backtest is tested against the losers, not just today's survivors;
- **announce-ahead** — a name is added on its *effective* date, not its announcement date, unless
  queried by announcement knowledge time.

## S&P 500

Reconstructed from the community-maintained `fja05680` S&P 500 dataset: a current-members table
plus a changes table of additions/removals back to 1996. Every ticker→CIK mapping is
EDGAR-verified (cover-page / full-text) or accepted on exact EDGAR-name identity — **no bare-ticker
guesses**.

### Coverage floor (honest, conservative)

Because a ticker-span is included only when verified, coverage is a **conservative floor that
thins the further back you go**. Approximate as-of-date constituent counts: 2024 ≈ 504, 2015 ≈ 495,
2008 ≈ 490, 2005 ≈ 487, 2000 ≈ 448 (near-complete from ~2005 on). Unresolved spans — predominantly
pre-2001 (EDGAR full-text can't index them, so cover-page verification can't run) plus a handful of
identity ambiguities — are **omitted, never guessed**. A queried pre-2010 date may therefore return
slightly fewer than 500 names; every name returned is verified-correct, but the set is not
guaranteed complete for older dates. See [`08-known-limitations.md`](08-known-limitations.md).

### Worked reference (golden test cases)

- **Lehman Brothers (LEH)**: a member on 2008-09-14; absent by 2008-09-22 (removal effective
  2008-09-16). The membership row's `valid_to` is the removal effective date.
- **Tesla (TSLA)**: absent on 2020-12-20, present on 2020-12-21 (inclusion effective date).

## DJIA

Survivorship-free as-of-date membership for the Dow Jones Industrial Average (`DJIA`), reconstructed
from the Wikipedia [Historical components of the Dow Jones Industrial Average](https://en.wikipedia.org/wiki/Historical_components_of_the_Dow_Jones_Industrial_Average)
change chronology (retrieved 2026-07-12), with recent ticker confirmation against yfiua monthly
snapshots. Source token on stored rows: **`wikipedia-djia+efts`**. Same reconstruction and
identity rules as SP500 (era-correct tickers; EDGAR-verified ticker→CIK where possible).

**Policy B (identity):** served constituents may carry `cik=null` for pre-EDGAR / unresolved
identity when the membership span is still sourced. Prefer resolved CIKs whenever EDGAR /
`company_tickers` can verify. **Never invent CIKs.** This is stricter on inventing mappings than
on omitting CIK; under-include membership only when the change log itself cannot support a span.

**Not an official S&P DJI product** — research reconstruction from public sources; do not treat as
a licensed official constituent feed.

### Coverage floor

The modern DJIA has **30** members; as-of counts are **near-complete for recent decades** and may
under-count where the free change log is incomplete (especially very deep history or ambiguous
ticker mappings). **Measured as-of counts (2026-07-12, production DB):**

| as_of_date | constituents |
|---|---:|
| 2000-06-30 | 29 |
| 2005-06-30 | 30 |
| 2008-09-15 | 30 |
| 2015-03-19 | 30 |
| 2020-06-30 | 30 |
| 2024-06-30 | 30 |

Do **not** claim exactly 30 names on every historical date. Coverage floor for modern PIT use is
essentially full (29–30 from ~2000 onward in the measured sample); earlier eras follow the Wikipedia
chronology and may include long-lived intervals whose CIK is null under Policy B when EDGAR cannot
verify.

### Worked reference (golden test cases)

- **Apple (AAPL)**: absent on 2015-03-18, present on 2015-03-19 (inclusion effective date; replaced
  AT&T / `T` the same day).
- **General Electric (GE)**: a member on 2018-06-25; absent on 2018-06-26 (removal effective date;
  replaced by Walgreens Boots Alliance / `WBA`) — survivorship-free long-tenured removal.

## Scope

Survivorship-free **as-of-date membership** is served for **S&P 500, DJIA, NDX, DJT, and DJU**.
Depth and source quality **differ by index** (SP500/DJIA are deepest; NDX is monthly modern-only;
DJT/DJU are modern-era floors — see sections below). Russell 3000 / broad-index multi-decade
membership remains a paid/hard data problem, out of scope.

## DJT (Dow Jones Transportation Average)

Survivorship-free as-of-date membership for the Dow Jones Transportation Average
(`DJT` / DJTA), reconstructed from the Wikipedia component table and prose change
notes (modern era ~2011→present). Same identity rules as SP500/DJIA (Policy B:
never invent CIKs). Depth is **shallower** than DJIA: no free full historical
change chronology. Measured as-of counts (2026-07-12): modern dates ≈ **20**
components; pre-2011 history is a conservative floor via open-start intervals.

## NDX (Nasdaq-100)

Survivorship-free as-of-date membership for the Nasdaq-100 (`NDX`), reconstructed from
free **monthly** constituent snapshots (yfiua/index-constituents, ~2023-07→present).
Same identity rules as other indices. **Depth is deliberately shallow**: there is no
reliable free multi-decade Nasdaq-100 change log suitable for redistribution. Add/remove
dates are the snapshot month when first observed, not necessarily the official rebalance
effective date. Measured as-of counts (2026-07-12): mid-2023 ≈ 100, 2024–2026 ≈ 100–101.

## DJU (Dow Jones Utility Average)

Survivorship-free as-of-date membership for the Dow Jones Utility Average (`DJU` / DJUA),
reconstructed from the Wikipedia component table and Former components prose (modern changes
~2014→present). Same identity rules as other Dow indices. Depth is modern-only; pre-2014
as-of queries under-include. Measured as-of counts (2026-07-12): 2014-10-01→present ≈ **15**.
