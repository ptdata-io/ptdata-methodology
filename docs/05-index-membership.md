# 05 — Index membership (historical S&P 500)

*Status: implemented (6e) from a reconstructed source (Wikipedia 'List of S&P 500 companies').*

## How it works (6e)

We parse the current-constituents table and the changes table (additions/removals with
dates), then **reconstruct** membership intervals by walking the change events backward from
the current snapshot: an "added on D" begins a membership at D; a "removed on D" ends one at
D; a current member with no relevant change has an unknown (pre-coverage) start and an open
end. Intervals are stored bitemporally in `index_membership` (valid time = membership span;
knowledge time = our reconstruction). A point-in-time query (`get_index_constituents(index,
as_of_date)`) returns the constituents as understood on that date — survivorship-free.

Validated against the golden cases on the live reconstructed source: Lehman (LEH) is a member
on 2008-09-14 and absent by 2008-09-22 (actual removal 2008-09-16); Tesla (TSLA) is absent on
2020-12-20 and present on 2020-12-21.

## Source and provenance

The free reconstruction (Wikipedia) is treated as a *source*, not an authority. Securities
are keyed by ticker; CIK linkage is later enrichment. Reconstruction quality is bounded by
the completeness of the change record — see limitations below and in
[`08-known-limitations.md`](08-known-limitations.md).

## Principle

S&P 500 membership is stored bitemporally in `index_membership`: valid time is the
membership span `[valid_from, valid_to)`; knowledge time captures the fact that changes are
announced ahead of their effective date and are occasionally corrected. See
[`01-bitemporal-model.md`](01-bitemporal-model.md).

## What this guarantees

- `get_index_constituents("SP500", as_of_date)` returns the membership as it was *understood
  on* `as_of_date` — surviving the two classic biases: survivorship (dropped names are still
  present for the dates they were members) and announce-ahead (a name is added on its
  effective date, not its announcement date, unless queried by announcement knowledge).

## Worked reference (golden test case)

- **Lehman Brothers (LEH)**: a member of the S&P 500 on 2008-09-14; removed by 2008-09-22.
  The membership row's `valid_to` is the removal effective date.
- **Tesla (TSLA)**: inclusion on 2020-12-21.

## To document here

The reconstructed source of historical constituents and its provenance; reconciliation
method; how index methodology changes (e.g. float adjustments) are or are not represented;
known coverage boundaries of the reconstruction.
