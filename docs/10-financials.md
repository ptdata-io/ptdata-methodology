# 10 — Financials: the as-reported revenue family

Banks, broker-dealers, and insurers do not have "revenue" the way an industrial does. An
industrial has one top line — *Net sales* / *Total revenues* — above cost of sales. A bank's
income statement instead presents **net interest income** and **noninterest income** as separate
subtotals, often with *no* grand-total revenue line at all; a broker-dealer presents *Total net
revenues* (revenues net of interest expense); an insurer presents *Premiums earned* and *Net
investment income*. Forcing a single "revenue" onto these filers is the classic data-vendor error:
the most common wrong answer is to report a bank's **Total interest income** as its revenue, which
silently omits all fee and trading income and is not a figure the bank itself treats as revenue.

ptdata refuses that synthesis. For financials we serve an **as-reported revenue family**: each
member is a labeled subtotal taken verbatim from the consolidated income statement. We never sum the
members into a fabricated total, and we never substitute one for "revenue." A financial is *covered*
when it reports **at least one** family member — not when any single metric is present.

## The revenue family

| Logical metric | us-gaap tag (XBRL) / as-reported line (pre-XBRL) | Typically reported by |
|---|---|---|
| `total_revenue` | `Revenues` — a literal *"Total revenues"* grand total | insurers, asset managers, banks that print a total |
| `total_net_revenue` | `RevenuesNetOfInterestExpense` — *"Total net revenues"* | broker-dealers (Goldman, Morgan Stanley) |
| `net_interest_income` | `InterestIncomeExpenseNet` — *"Net interest income"* | banks |
| `noninterest_income` | `NoninterestIncome` — *"Noninterest income"* | banks |
| `premiums_earned` | `PremiumsEarnedNet` — *"Premiums earned, net"* | insurers |
| `net_investment_income` | `NetInvestmentIncome` — *"Net investment income"* | insurers |

`total_revenue` is the *Total revenues* grand total where a filer prints one; it resolves to the same
underlying `Revenues` concept as the general `revenue` metric. The general **`revenue`** metric is
*additive and unchanged*: where a financial does print a single total (most insurers, Schwab, many
bank holding companies), `revenue` still resolves to it. The family is what we serve **in addition**,
so that a filer with no single total (Goldman) is still fully covered.

REITs are **not** treated as financials: they report real rental revenue, which lands on the ordinary
`revenue` metric. (`RealEstateRevenueNet` is mapped in the XBRL tier for REITs that tag only that.)

## Worked examples (as reported, fiscal 2023)

These are queryable and falsifiable against the filings:

- **JPMorgan (`JPM`)** — a bank that *also* prints a total: `total_revenue` $158.1B,
  `net_interest_income` $89.27B, `noninterest_income` $68.84B.
- **Bank of America (`BAC`)** — `total_revenue` $98.58B, `net_interest_income` $56.93B,
  `noninterest_income` $41.65B.
- **Goldman Sachs (`GS`)** — a broker-dealer with **no** *Total revenues* line: `total_net_revenue`
  $46.25B, `net_interest_income` $6.35B, `noninterest_income` $39.9B. Asking `GS` for `revenue`
  returns *guidance*, not silence (below).
- **MetLife (`MET`)** — an insurer: `total_revenue` $66.91B, `premiums_earned` $44.28B,
  `net_investment_income` $19.91B.

The members a given filer does not report are **absent**, not zero — a bank has no
`premiums_earned`; an insurer has no `noninterest_income`. That is correct, as-reported structure.

## What we deliberately do *not* do

- **No derivation.** We never compute a bank "revenue" as `net_interest_income + noninterest_income`,
  nor net a broker's gross revenues ourselves. Every served number is a line the filer printed.
- **No forced cross-company comparability.** Normalizing a bank, a broker, and an insurer onto one
  "revenue" axis is an opinionated modeling choice that belongs to the consumer, not the data. We
  serve the as-reported structure and let the consumer decide.
- **No silent single revenue.** Where a filer prints no total, `revenue` is genuinely absent and we
  say so (with a pointer to the family), rather than quietly returning a partial line as if it were
  the whole.

We *do* preserve **cross-era** consistency: each named metric maps to the same as-reported line in
2003 and 2023, across the pre-XBRL→XBRL seam (below).

## Both tiers, in lockstep (era-symmetry)

The family is served identically in the XBRL tier (FY2009+) and the pre-XBRL parsed tier
([`09-pre-xbrl-fundamentals.md`](09-pre-xbrl-fundamentals.md)). The same six logical metrics resolve
to the same concepts on both sides of the ~2009 XBRL floor, so a bank's `net_interest_income` series
is continuous across it — no seam where the metric silently changes meaning.

In the parsed tier, extraction is **sector-aware**: a financial is extracted on a *financial profile*
whose `revenue` instruction is strict — *only* a line literally labeled "Total revenues" (else null,
never "Total interest income") — plus the five family members; it drops the metrics financials don't
report (operating income, EBIT). This is the correctness mechanism that prevents the
interest-income-as-revenue mis-map at the source. Industrials keep the standard 12-metric profile.

## Which filers are "financial"

The profile is selected from the filer's SEC **SIC** code (persisted per company):

| SIC | Class | Treatment |
|---|---|---|
| 6000–6199, 6712 | depository / credit / **bank holding companies** | financial |
| 6200–6299 | security & commodity **brokers/dealers** | financial |
| 6300–6499 | **insurance** carriers & agents | financial |
| 6726 | investment offices / funds | financial |
| **6798** | **REITs** | *not* financial (real rental revenue) |
| 6500s, 6770, … | real estate, other | *not* financial |

Note `6712` (bank holding companies) — these sit outside the 6000–6499 band and are explicitly
included, since they are banks in substance.

## Coverage (measured)

Against the live database, of operating financials with XBRL facts, **~99% report ≥1 family member**
(banks ~98%, insurers ~100%, brokers ~99% of operating brokers). The residual is *structural, not a
missing tag*:

- **Revenue-less investment vehicles** (SIC 6221 commodity/crypto ETF trusts — Grayscale, iShares
  Bitcoin, etc.): these have no revenue concept of any kind; their income is gains/losses on the
  underlying holdings (`NetIncomeLoss` only). Absence is correct.
- **Foreign private issuers** filing under IFRS tags (`ifrs-full:Revenue`) rather than us-gaap: a
  separate tagging limitation, not a revenue-family gap.
- A small tail of SPAC/BDC shells and holding-LP structures (no consolidated revenue line).

See [`08-known-limitations.md`](08-known-limitations.md) and the server-side
`DATA_LIMITATIONS.md` registry for the itemized list.

## What the API returns for an unavailable metric

Because financials genuinely lack some metrics, `get_fundamentals` never *silently* omits a requested
metric. Anything with no value appears under **`unavailable`**, keyed by the requested metric name,
with structured guidance:

```jsonc
// get_fundamentals("GS", as_of_date=..., metrics=["revenue"])
{
  "metrics": { /* … whatever resolved … */ },
  "unavailable": {
    "revenue": {
      "reason": "GS is a broker (SIC 6211) and does not report a single 'revenue' line. Financials report revenue as an as-reported FAMILY of subtotals, never a synthesized total. Use available_alternatives.",
      "available_alternatives": ["total_net_revenue", "net_interest_income", "noninterest_income"],
      "docs_url": "https://github.com/ptdata-io/ptdata-methodology/blob/main/docs/10-financials.md"
    }
  }
}
```

`available_alternatives` lists the metrics that **actually have data** for that entity and period
(for a revenue request on a financial, the populated family members), so an agent can immediately
re-query the right thing. The same mechanism applies to any unavailable metric on any filer (e.g.
`operating_income` on a single-step filer) — guidance instead of silence.

## Principle

ptdata serves the **as-reported structure** of a financial's income statement, not a forced single
revenue. The product's job is to hand back exactly what the filer reported, labeled and with lineage;
turning a bank, a broker, and an insurer into one comparable "revenue" number is a modeling decision
we leave to the consumer. See [`03-fundamentals.md`](03-fundamentals.md) for the general
as-reported / point-in-time model and [`01-bitemporal-model.md`](01-bitemporal-model.md) for the
knowledge-time guarantees that apply equally to every metric here.
