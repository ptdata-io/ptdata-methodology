# 01 — The bitemporal model

This document describes the conceptual model. The **canonical, executable source of truth**
is the DDL and Alembic migrations in the private `ptdata-server` repository
(`storage/schema.sql` and `storage/migrations/`). Where this prose and the migrations ever
disagree, the migrations win, and this document is the bug. The DDL is reproduced inline
below for readability, but it is a copy.

## 1. Two time axes

Every fact in ptdata is positioned on two independent time axes:

- **Valid time** — the interval during which the fact is true in the modeled world.
  - Period facts (a fiscal-period fundamental, an index-membership span) use a half-open
    interval `[valid_from, valid_to)`.
  - Daily-grain facts (a day's price, a day's option chain, an implied-expectation
    observation) use a single `valid_date`. A point is just the degenerate interval
    `[valid_date, valid_date + 1 day)`; we store it as a date for ergonomics.
- **Knowledge time** — the interval during which *our store* held this version as current
  knowledge: `[knowledge_from, knowledge_to)`, with `knowledge_to = 'infinity'` for the
  currently-believed version. Knowledge time is system-managed and monotonic.

A fact version is therefore a rectangle in the (valid, knowledge) plane. A point-in-time
query is a point in that plane.

## 2. The point-in-time query

Every read answers *"as of knowledge date K, what was true for valid coordinate V?"*:

```sql
-- period-grain facts
WHERE valid_from      <= V AND V < valid_to
  AND knowledge_from  <= K AND K < knowledge_to

-- daily-grain facts
WHERE valid_date       = V
  AND knowledge_from  <= K AND K < knowledge_to
```

With correct maintenance (§4) this returns exactly one version per logical fact. `K`
defaults to `now()` only when the caller explicitly wants the latest knowledge; the API
requires an explicit `as_of_date` for any query whose correctness depends on it.

## 3. Immutability and append-only ingestion

We never overwrite a value. A new observation of an already-known fact is a **new row**.
The only in-place write the system ever performs is *closing* a superseded version's
`knowledge_to` from `'infinity'` to the timestamp at which the newer version became known.
This is a temporal bookkeeping operation: it changes when a version *was believed*, never
*what it said*. The value columns of an existing row are immutable for the row's life.

> **Design note (honest trade-off).** A maximally strict design would keep an
> insert-only event log with no in-place close at all, deriving `knowledge_to` at query
> time via window functions. We chose the closed-interval model because it turns every
> point-in-time read into a single indexed range predicate (§2) instead of a windowed
> scan, at the cost of one bookkeeping `UPDATE` per supersession. Full history is equally
> reconstructable either way; nothing is destroyed. We consider this the right
> correctness/performance balance and document it rather than hide it.

## 4. Maintenance algorithm (supersession)

To record that, at knowledge time `K2`, the value for a logical fact (identified by its
business key + valid coordinate) is now `X2`:

1. `UPDATE` the current version (the one with `knowledge_to = 'infinity'` and matching
   business key + valid coordinate) to set `knowledge_to = K2`.
2. `INSERT` a new row with the new value, `knowledge_from = K2`, `knowledge_to = 'infinity'`,
   and the source/lineage of the new observation.

Both statements run in one transaction. If the incoming observation is value-identical to
the current version, it is a no-op (idempotent re-ingestion).

## 5. Integrity constraints

For each bitemporal fact table, a GiST exclusion constraint (via the `btree_gist`
extension) forbids two versions of the *same logical fact* from having overlapping
knowledge intervals. This makes "exactly one version is current for any knowledge
coordinate" a database-enforced invariant, not a convention. Example for `financial_fact`:

```sql
EXCLUDE USING gist (
    company_id  WITH =,
    concept_id  WITH =,
    valid_range WITH &&,          -- tstzrange(valid_from, valid_to)
    knowledge_range WITH &&       -- tstzrange(knowledge_from, knowledge_to)
)
```

## 6. Table catalog

**Identity & reference**

- `company` — issuer identity; `cik` (SEC Central Index Key) is the natural key.
- `security` — a tradable security belonging to a company.
- `security_identifier` *(bitemporal)* — maps a security to its tickers/CUSIPs over time.
  Bitemporal because identifiers are reassigned (e.g. `FB` → `META`, 2022-06-09); a
  point-in-time query must resolve the ticker that was valid on the date in question.
- `financial_concept` — dictionary normalizing source tags (e.g. `us-gaap:Revenues`) to
  ptdata metric names; carries the unit type and definition.
- `methodology_version` — registry of methodology versions (semver + git SHA + notes).
  Every *derived* fact references the version under which it was computed.
- `ingestion_run` — audit row per ingestion job (source, window, status), referenced as
  lineage by ingested facts.

**Filings & lineage**

- `filing` — one row per SEC submission, keyed by `accession_number`. Holds `cik`,
  `form_type`, `filing_date`, `acceptance_datetime` (EDGAR's exact acceptance time),
  `period_of_report`, document URL, and `amends_accession` (self-reference) which builds
  amendment/restatement chains. This table is the lineage backbone: every derived fact
  points at the accession it came from.

**Facts** *(all bitemporal unless noted)*

- `financial_fact` — as-reported financial values. Valid time = the fiscal period; a
  restatement is a new version with later knowledge time. Lineage → `source_accession`.
- `price_raw` — daily OHLCV. Immutable raw vendor data; bitemporal on knowledge so vendor
  *corrections* are versioned without mutating the original print.
- `corporate_action` — splits, dividends, spinoffs. Bitemporal because actions are
  announced ahead of, and occasionally corrected after, their ex-date. Drives price
  adjustment (§7).
- `index_definition` / `index_membership` — index identity and bitemporal membership
  spans. Valid time = membership interval; knowledge time captures announce-ahead and
  corrections.
- `earnings_event` — links an 8-K (`filing_accession`) to a fiscal period, recording
  `sec_acceptance_ts` (exact, from EDGAR) and a nullable best-effort `dissemination_ts`
  (press-release/wire time — the real market event) with source + confidence. As-reported
  figures are the `financial_fact` rows sourced from the same 8-K.
- `options_chain_eod` — historical end-of-day option chains (per strike/expiration:
  bid/ask/last, volume, open interest, IV, greeks). Immutable; the raw input to derived
  expectations.
- `implied_expectation` *(derived, bitemporal)* — per security per observation date: the
  options-implied expected move, ATM implied volatility, optional risk-neutral-distribution
  parameters, the `method` and `methodology_version`, and lineage to the source chain
  snapshot.

## 7. Point-in-time corporate-action adjustment (no raw mutation)

`price_raw` is never rewritten by a split. `get_price_history(ticker, …, adjustment_mode,
as_of_date)` computes a cumulative adjustment factor from the `corporate_action` rows whose
`knowledge_from <= as_of_date`, and applies it to the raw series. Consequences:

- The adjusted series is a *function of knowledge time*: a split known only after
  `as_of_date` does not retroactively adjust a backtest's view of earlier prices.
- The response returns raw price, adjusted price, the applied factor, and the
  corporate-action lineage — the adjustment is reproducible and auditable.

## 8. Indexing strategy

The dominant access pattern is *"business key + valid coordinate, as known on K."*

- **Latest-known-≤-K:** composite btree on `(business_key…, valid_*, knowledge_from DESC)`
  per fact table (e.g. `financial_fact(company_id, concept_id, valid_from, knowledge_from
  DESC)`), so the current-or-historical version is the first index row.
- **Current view (hot path):** partial index `… WHERE knowledge_to = 'infinity'`.
- **Range-contains & integrity:** GiST on `tstzrange(valid_*)` and `tstzrange(knowledge_*)`
  (`btree_gist` for the composite exclusion constraints in §5).
- **Earnings/options:** btree `(security_id, observation_date)` on `implied_expectation`
  and `(security_id, trade_date, expiration)` on `options_chain_eod`.

## 9. Lineage in every response

Per commitment #3, each value an API returns is accompanied by: its `source` (accession
number, vendor revision id, or chain-snapshot id), its `knowledge_from` (when we learned
it), and the `methodology_version` in force. The Pydantic response models (defined before
tool logic, per commitment #4) make these fields non-optional.

---

## Appendix A — DDL (copy of `ptdata-server/storage/schema.sql`)

See the private repository for the canonical, migration-managed version. Reproduced here so
this document is self-contained; treat the migrations as authoritative.
