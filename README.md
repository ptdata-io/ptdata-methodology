# ptdata — methodology

**Point-in-time historical market data for US equities. Backtest with the data that was
actually available at the time — not the data as it looks today.**

This repository is the public, falsifiable specification of how the ptdata service works.
Trust in a point-in-time data product cannot come from marketing; it has to come from
*falsifiability*. So the methodology is open, and a companion repository
([ptdata-correctness](https://github.com/ptdata-io/ptdata-correctness)) contains a runnable
test suite that anyone can execute against the live service
([api.ptdata.io](https://api.ptdata.io)) to check that it behaves as documented. If we are
wrong, the tests fail, and you can see it.

## The problem: lookahead bias

Almost every affordable market-data API serves you the world *as it looks today*. When
you ask for Apple's FY2017 revenue, you get the number after all subsequent restatements.
When you ask which companies were in the S&P 500 in 2008, you get a list shaped by who
survived. When you reconstruct a price series across a split, you get prices adjusted by
splits that *hadn't happened yet* on the date you're backtesting.

Each of these silently leaks future information into the past. A backtest built on that
data is testing a strategy that had access to facts no one had at the time. The results
are optimistic, and the optimism is invisible — which is the dangerous kind. Institutional
desks pay $20K+/year (CRSP, Compustat, Refinitiv) precisely to avoid this. ptdata brings
the same point-in-time correctness to a price point and a distribution model accessible to
independent quants, event-driven traders, and small funds.

## The solution: bitemporal storage

ptdata records two timestamps for every fact: **valid time** (when the fact was true in
the world) and **knowledge time** (when *we* learned it). Every query answers a precise
question — *"what did we know on date K about valid time V?"* — instead of *"what is true
now?"* Restatements, corporate-action corrections, and index changes are appended as new
versions, never destructive overwrites, each linked to its source (an SEC accession number,
a vendor revision). Full lineage travels in every API response. See
[`docs/01-bitemporal-model.md`](docs/01-bitemporal-model.md).

## Earnings events: priced-in expectation, not consensus

Our earnings feature is built on the expectation the **options market** had priced in
immediately before each announcement — the implied expected move — rather than analyst
consensus. This is a deliberate choice. Genuine point-in-time analyst consensus is an
institutional-only data product whose licensing forbids redistribution; serving a cheap
imitation would be exactly the kind of quietly-wrong data this product exists to replace.
The options-implied expected move is point-in-time-correct *by construction* (an option's
price on a date is simply what it was), it is the directly tradeable expectation for an
event-driven strategy, and it is built from public-domain filing timestamps plus historical
options analytics. We never label it "consensus." See
[`docs/06-earnings-events.md`](docs/06-earnings-events.md) and
[`docs/07-options-implied-expectations.md`](docs/07-options-implied-expectations.md).

## Coverage status

**Live public API:** [https://api.ptdata.io](https://api.ptdata.io) (free IP-limited tier + paid
Pro/Scale via account). Measured coverage and every known gap are tracked in
[`docs/08-known-limitations.md`](docs/08-known-limitations.md) — read it before relying on anything
here. Product site: [https://ptdata.io](https://ptdata.io).

- **Universe:** all EDGAR (~8,300 companies) for fundamentals & filings; survivorship-free **index
  membership** for **S&P 500, DJIA, NDX, DJT, and DJU** (depth varies by index — NDX is monthly-grain
  and modern-only; see [`docs/05-index-membership.md`](docs/05-index-membership.md)).
- **Fundamentals:** full statements (income, balance sheet, cash flow) for **FY2009→
  present** (the SEC XBRL era), plus three XBRL-only share counts (`shares_outstanding`,
  `weighted_avg_shares_basic`, `weighted_avg_shares_diluted`), plus a clearly-labeled **parsed pre-XBRL tier** for earlier periods
  (growing curated/batch set); banks/brokers/insurers get an as-reported **revenue family**
  ([`docs/10-financials.md`](docs/10-financials.md)). Earnings (8-K timestamps) span the full filing
  history. Fundamentals are kept fresh via **daily EDGAR incremental** ingestion. **Source of
  record:** SEC EDGAR (public domain).
- **Raw prices are out of scope for the redistributable product** — by design. Prices are a
  commodity with no public-domain source and licensing that forbids cheap redistribution; ptdata
  serves the point-in-time *corporate* data nobody else does, and leaves prices to dedicated
  (often free, for personal use) price sources. We *do* serve **derived** event analytics
  (e.g. a realized earnings reaction as a return) where they're differentiated. Options-implied
  expected move is specified but **not yet live** (see limitations). See
  [`docs/08-known-limitations.md`](docs/08-known-limitations.md).

## Repository layout

| Doc | Contents |
|---|---|
| [`00-overview.md`](docs/00-overview.md) | Architecture and the core claims, with worked examples |
| [`01-bitemporal-model.md`](docs/01-bitemporal-model.md) | Valid time vs. knowledge time; the schema (source of truth) |
| [`02-edgar-ingestion.md`](docs/02-edgar-ingestion.md) | Filings: ingestion, versioning, accession-chain linking |
| [`03-fundamentals.md`](docs/03-fundamentals.md) | As-reported vs. restated handling |
| [`04-corporate-actions.md`](docs/04-corporate-actions.md) | Point-in-time price adjustment and lineage |
| [`05-index-membership.md`](docs/05-index-membership.md) | Historical S&P 500 constituents |
| [`06-earnings-events.md`](docs/06-earnings-events.md) | 8-K timestamping, as-reported figures, event windows |
| [`07-options-implied-expectations.md`](docs/07-options-implied-expectations.md) | Options-implied expected move; data source; licensing basis |
| [`08-known-limitations.md`](docs/08-known-limitations.md) | Explicit caveats and coverage gaps |
| [`09-pre-xbrl-fundamentals.md`](docs/09-pre-xbrl-fundamentals.md) | The parsed tier: pre-2009 fundamentals, confidence index |
| [`10-financials.md`](docs/10-financials.md) | Banks/brokers/insurers: the as-reported revenue family |

## License

Documentation in this repository is licensed [CC-BY-4.0](LICENSE). The correctness test
code is licensed MIT in its own repository.
