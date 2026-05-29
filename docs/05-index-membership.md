# 05 — Index membership (historical S&P 500)

*Scaffold — to be completed alongside constituents ingestion (build step 6e).*

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
