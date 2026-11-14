# Methodology v0.1 — initial release

**Status:** pre-release / in development.

## Established in this version

- **Bitemporal model** (valid time + knowledge time), append-only with knowledge-interval
  close-out. See `docs/01-bitemporal-model.md`.
- **Schema** as source-of-truth in the Alembic migrations / `storage/schema.sql`; conceptual
  model documented in `docs/01`.
- **Earnings events** built on options-implied expected move, not analyst consensus, with a
  two-timestamp model (`sec_acceptance_ts` + best-effort `dissemination_ts`). See `docs/06`,
  `docs/07`.
- **Point-in-time corporate-action adjustment** (raw prices never mutated). See `docs/04`.

## Key decision recorded

ptdata serves **no analyst consensus**. Vendor-licensing research established that
cheap + redistributable + point-in-time consensus does not exist; the options-implied
expected move is served instead as the (point-in-time-correct-by-construction) earnings
expectation benchmark. Rationale: `docs/07-options-implied-expectations.md`.

## Open before launch

- Confirm OPRA historical redistribution carve-out with OPRA/counsel and chosen vendor terms.
- Fix daily-price vendor and confirm its redistribution terms.
- Establish dissemination-time sourcing for earnings events.

(See `docs/08-known-limitations.md`.)

## Added while still pre-release (2026-11-10)

- A missing Q4 flow value can be the annual total minus Q1, Q2, and Q3 of the same concept,
  computed when the request asks for a quarter that was not tagged. `provenance` is `derived`.
  `methodology_version` stays `0.1.0`.
- Q2 and Q3 operating cash flow and capex stay the filed quarter when its duration is about
  three months. Otherwise they are the year-to-date difference.
- Logical metrics `cost_of_revenue` and `eps_basic`. EPS and share counts are not derived for Q4.
- Earnings-event `fiscal_period` in the tool text is FY, or Q1–Q3. A fourth-quarter release
  stays on FY. `fiscal_year` and `fiscal_period` are required, and there is no `as_of_date`.
- A quarter-length fact that ends on the fiscal year-end is that year's Q4, including when
  a later filing stamps its own period on the comparative column. The tagged amount is
  served. The annual figure is not subtracted.
