# 09 — Pre-XBRL fundamentals (the parsed tier)

SEC structured XBRL (`companyfacts`) begins ~2009. For periods before that — and for companies
that died before it (Enron, WorldCom, Lehman) — fundamentals can only be obtained by reading the
filing text. ptdata serves a growing, **clearly-labeled, lower-confidence tier** of such values,
extracted from the original filings. It is *additive* to the XBRL core and never replaces it.

## What it is
- **Metrics:** the same 12 logical metrics as the XBRL tier (full income statement + balance sheet +
  cash flow), plus the financials revenue family for banks/brokers/insurers
  ([`10-financials.md`](10-financials.md)).
- **Scope:** a *curated* set of high-value companies (notable defunct issuers + S&P stalwarts for
  boundary continuity) **plus a growing prioritized batch** (S&P-500-historical names first) —
  **~1,500+ companies so far** (growing; measured on production), not yet the full pre-XBRL
  universe. Annual (10-K) is the default; specific flagship quarters (10-Q) are included where
  they back a golden case.
- **Labeling:** every value carries `provenance` and `extraction_confidence` (below). XBRL values
  are `provenance="xbrl"` with no extraction confidence; parsed values are `provenance="parsed"`.

## How values are extracted
Values come from a **fault-tolerant, multi-agent pipeline** run one company at a time, per fiscal
year. There is no separate "locate" step that can fail closed: **two independent agents
self-navigate** the filing (`grep` to the *consolidated* statements — rejecting the table of
contents, the auditor's report, Schedule-I parent-company-only and segment/combined statements, and
the separately-presented statements of a consolidated subsidiary), and each **extracts every metric**
(full income statement, balance sheet, and cash flow), citing the exact source line and the stated
scale ("in millions/thousands"; the year-column header so column *order* is irrelevant). An
**adversarial verifier** then tries to *refute* each value — right statement? right line? right
fiscal-year column? right scale/sign (capex positive, balances as-of the period end, EPS per-share)?
The two extractions and the verifier loop up to three times on any metric that hasn't converged, and
a **deterministic table parser** supplies a corroborating signal where the layout permits. The
combination — independent agreement, then adversarial verification — is the correctness gate
(below). Extraction is **sector-aware**: financials (banks/brokers/insurers) run a profile with a
strict revenue rule + the as-reported revenue family, so a bank's "Total interest income" is never
mis-read as revenue (see [`10-financials.md`](10-financials.md)). These agents run offline/batch,
never in the live service.

## Confidence index

Every parsed fact carries an **`extraction_confidence`** label, returned on every API/MCP response
(alongside `provenance`) so consumers can filter or weight by trust. It reflects how many
independent signals (the deterministic table parser, two independent extraction agents) agreed and
whether the adversarial verifier confirmed:

| `extraction_confidence` | Meaning | Published? |
|---|---|---|
| **`verified`** | All three signals (deterministic parser + both extract agents) agree **and** the verifier confirms — or the value was hand-checked against the primary filing. | yes |
| **`high`** | Two independent agent extractions agree **and** the adversarial verifier confirms. | yes |
| **`medium`** | A value the adversarial verifier confirmed against the filing, adjudicated across both extractions (so a value one extractor missed can still qualify), where the two extractions did not independently agree; magnitude-bounded. | yes |
| **`low` / unresolved** | Signals conflicted, or the verifier refuted, or no convergence. | **no — flagged, withheld** |

XBRL-sourced facts are labeled `provenance='xbrl'` with **no** `extraction_confidence` (they are not
extracted from text). Only parsed facts carry the index. A fact is published only at `verified`,
`high`, or `medium`; nothing reaches the data set without independent agreement + verification.

## Integration rules (point-in-time + lineage preserved)
- **Gap-fill only — XBRL always wins.** A parsed fact is written only when no XBRL fact exists for
  that (company, concept, fiscal_year, fiscal_period). The two tiers never collide.
- **Bitemporal + lineage unchanged:** `knowledge_from` = the filing's acceptance time;
  `source_accession` = the originating 10-K/10-Q. A point-in-time query returns the value known on
  the as-of date, exactly as for XBRL facts.
- **Same query surface:** parsed facts are returned by `get_fundamentals` alongside XBRL facts,
  distinguished only by the `provenance` / `extraction_confidence` fields.

## Current coverage (measured)
**Production (2026-07-23):** ~**1,539 companies** with current parsed facts, ~**103,781** current
parsed facts (confidence: ~1,681 `verified`, ~99,376 `high`, ~2,724 `medium`). Coverage grows
company-by-company through prioritized offline batches (S&P-500-historical names first); operators
track progress in the private `ptdata-server` repo (`PREXBRL_PROGRESS.md` / batch runs). Counts
change as batches ship to production — treat these as order-of-magnitude, not a frozen guarantee.
The initial curated set includes:
- **Enron** (CIK 0001024401) Q3 2001 — revenue $46,877M, net income $(644)M — `verified`.
- **WorldCom/MCI** (CIK 0000723527) Q1 2002 — *consolidated* revenue $8,120M, net income $172M
  (original pre-fraud-disclosure figures, not the MCI-group subtotal) — `verified`.
- **Exxon Mobil** (CIK 0000034088) FY2004–2008 — sales-and-other-operating-revenue basis (matching
  us-gaap:Revenues from FY2009); net income `high`/`verified`, revenue `medium`. Continuity series
  onto XBRL FY2009 ($301.5B), same basis.
- **Bear Stearns** (CIK 0000777001) FY2004–2007 — gross *Total revenues* ($8.4B→$16.6B) + net
  income ($1.34B→$2.05B→$233M); the consolidated statement, NOT the Schedule-I parent-only one.
- **General Motors** (old, CIK 0000040730) FY2004–2008 — *Total net sales and revenues*
  ($193B→$149B) + net income incl. the −$38.7B (FY2007) and −$30.9B (FY2008) losses.
- **General Electric** (CIK 0000040545) FY2004–2008 — *Total revenues* ($152B→$182.5B) + net
  earnings attributable to the Company ($16.6B→$17.4B); continuity onto XBRL FY2009 ($154.4B).
- The Enron + WorldCom facts back the `enron_2001_q3_10q` and `worldcom_2002_10q` correctness cases.

Beyond the curated names, a **prioritized batch** (S&P-500-historical first, FY2001–2008, same gated
pipeline) extracts the **full 12-metric statements** per company-year and now accounts for the bulk
of the 329-company tier — e.g. Abbott, AMD, Air Products, Skyworks, Howmet/Alcoa, AEP, Coca-Cola,
Colgate, Cummins, Corning, Target, Delta, and many more. The live, regenerated list is in
`ptdata-server/PREXBRL_PROGRESS.md`.

**Reliable-era floor (FY2001).** The parsed tier extracts fiscal years **2001–2008** only — the
window where consolidated-statement HTML is reliably machine-readable, ending where XBRL takes over
(2009). A consequence for S&P-historical coverage: a handful of index members whose *last* 10-K
predates FY2001 have **no in-scope filing to parse** and are therefore structurally out of scope, not
a coverage gap. These are companies acquired or merged during 2001 (e.g. Quaker Oats → PepsiCo,
ALZA → J&J, American General → AIG, Tosco → Phillips, Homestake Mining → Barrick). Their newest
10-K is FY2000 or earlier, so there is nothing to extract in the covered window. Progress metrics
count only in-scope names in the denominator so the ratio is not misleadingly short; the excluded
below-floor CIKs are reported separately. (Defunct issuers that *did* file into 2001 — Enron,
WorldCom — are covered via their in-scope quarterly filings; see above.)

## As reported, not normalized — discontinuities to understand
Parsed values are **exactly as stated in the original filing**, keyed by SEC **CIK** with accession
lineage. They are deliberately **not** restated to be comparable across two real-world breaks:
- **Entity identity vs the current ticker.** A ticker resolves to a CIK's *current* name, but the
  CIK's pre-2009 filings may be a predecessor's. E.g. ticker `HWM` (Howmet) → CIK 4281, whose
  pre-2009 figures are **Alcoa Inc's** (~$20–31B revenue); `SWKS` FY2001 is **Alpha Industries**;
  CIK 0000040730 (`GM`/"Motors Liquidation") is the *old* General Motors. The figures are correct
  for that CIK and filing; the ticker is just the current handle. *(Roadmap: surface the as-filed
  entity name alongside the ticker.)*
- **Accounting-basis changes.** A revenue or income series can step for presentation, not business,
  reasons. E.g. **AEP** revenue is $61.3B (FY2001, gross energy-trading) then $14.6B (FY2002) after
  the **EITF 02-3** gross→net reclassification; Bear Stearns is shown gross, not net of interest;
  Skyworks changed its fiscal year-end (March→September) in 2002, so FY2001 and FY2002 are not a
  continuous 12-month series. Each value is the correct as-reported figure for its filing.

**Operational notes.** A fact is **auto-loaded only when ≥2 independent signals agree AND the
verifier confirms** (`verified` if the deterministic parser also agrees, else `high`); anything else
is **flagged for human review and never written**. The extraction agents run interactively on the
operator's Claude subscription (no API key); a standalone `claude_extract` path (for headless
batches) is the only one needing `ANTHROPIC_API_KEY`. The line-based deterministic parser fails on
several pre-2009 formats (HTML one-cell-per-line; broker Schedule-I; multi-segment) — which is
exactly why the two self-navigating extract agents carry the extraction and the adversarial verifier
guards against the wrong-statement trap (a Schedule-I parent-only or an embedded subsidiary's
statements read instead of the registrant's consolidated ones — e.g. Conrail inside a CSX 10-K).

## Known limitations of this tier
- **Curated, not comprehensive.** Only the curated companies/periods are covered; absence of a
  pre-2009 value is expected, not a guarantee of non-existence.
- **Extraction has an error profile.** Unlike XBRL (machine-tagged at the source), these values are
  read from unstructured text. Every served fact is labeled by `extraction_confidence`
  (`verified` / `high` / `medium`); facts that fail the gate (no ≥2-signal agreement, or the
  adversarial verifier refutes) are flagged and **withheld**, never auto-loaded.
- **`valid_range` is a calendar-year approximation** for annual parsed facts (the authoritative key
  is `(fiscal_year, fiscal_period)`; the date range is informational). Non-calendar fiscal years
  are therefore approximate in `valid_range` only.
- **No restatement chains within the parsed tier** (rare pre-XBRL restatements are not modeled as
  bitemporal versions here; the point-in-time guarantee holds against the original filing).
- **No Compustat/vendor data** — this tier is parsed from public-domain EDGAR filings only, so it
  remains redistributable. See [`08-known-limitations.md`](08-known-limitations.md).
