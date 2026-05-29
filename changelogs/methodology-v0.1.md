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
