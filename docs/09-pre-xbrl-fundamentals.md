# 09 — Pre-XBRL fundamentals (the parsed tier)

SEC structured XBRL (`companyfacts`) begins ~2009. For periods before that — and for companies
that died before it (Enron, WorldCom, Lehman) — fundamentals can only be obtained by reading the
filing text. ptdata serves a small, **clearly-labeled, lower-confidence tier** of such values,
extracted from the original filings. It is *additive* to the XBRL core and never replaces it.

## What it is
- **Metrics:** revenue and net income only (the same logical metrics as the XBRL tier).
- **Scope (Phase 1):** a *curated* set of high-value companies — notable defunct issuers plus a
  few S&P stalwarts for boundary continuity — not the full pre-XBRL universe. Annual (10-K) is
  the default; specific flagship quarters (10-Q) are included where they back a golden case.
- **Labeling:** every value carries `provenance` and `extraction_confidence` (below). XBRL values
  are `provenance="xbrl"` with no extraction confidence; parsed values are `provenance="parsed"`.

## How values are extracted
The financial-statements region of the primary 10-K/10-Q is isolated, then revenue + net income
are obtained from up to three independent signals and reconciled:
1. **Deterministic table parse** — reads the statement's scale ("in millions/thousands"), the
   year-column header (so column *order* is irrelevant), and the exact line labels ("Total
   revenues" / bottom-line "Net Income", not "Net income before…"). For quarterly statements that
   show both three-month and nine-month columns, the three-month (the quarter) is taken.
2. **Claude-assisted extraction** — the located statement text is read by an LLM that returns the
   same fields as structured JSON (offline/batch only — never in the live service).
3. **MD&A prose** — the headline figures restated in the narrative ("net income for 2000 was
   $979 million…") as an independent check.

## Confidence index

Every parsed fact carries an **`extraction_confidence`** label, returned on every API/MCP response
(alongside `provenance`) so consumers can filter or weight by trust. It reflects how many
independent signals (the deterministic table parser, two independent extraction agents) agreed and
whether the adversarial verifier confirmed:

| `extraction_confidence` | Meaning | Published? |
|---|---|---|
| **`verified`** | All three signals (deterministic parser + both extract agents) agree **and** the verifier confirms — or the value was hand-checked against the primary filing. | yes |
| **`high`** | Two independent agent extractions agree **and** the adversarial verifier confirms. | yes |
| **`medium`** | A single extraction signal (no independent second could be obtained); magnitude-bounded and sanity-checked. | yes |
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
**12 companies, 138 facts** (confidence: 18 `verified`, 116 `high`, 4 `medium`; 0 flagged).
Coverage grows company-by-company through a prioritized, budget-capped batch (S&P-500-historical
names processed first) against a queue of ~2,238 pre-XBRL filers; current status is tracked in
`ptdata-server/PREXBRL_PROGRESS.md`. The initial curated set:
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

The prioritized batch (S&P-500-historical names first, FY2001–2008, same gated pipeline), 16 facts
each (revenue + net income):
- **Abbott** (ABT, CIK 1800), **AMD** (CIK 2488), **Air Products** (APD, CIK 2969), **Skyworks**
  (SWKS, CIK 4127), **Howmet/Alcoa** (HWM, CIK 4281), **American Electric Power** (AEP, CIK 4904).

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

**How the curated set was extracted (the fault-tolerant pipeline).** Bear Stearns / GM / GE / XOM
were run through a multi-agent pipeline (`ingestion/edgar/prexbrl/pipeline*.py` +
`workflows/prexbrl_extract.js`), one company at a time, per fiscal year: an agent **locates** the
correct consolidated statement (rejecting the table of contents, Schedule-I parent-only,
segment/group statements, and the balance sheet); **two independent agents extract** revenue + net
income (citing the exact source line + scale); an adversarial agent **verifies** (right statement?
right line? right year column? right scale/sign?), looping up to 3× on a refutation. A fact is
**auto-loaded only when ≥2 independent signals agree AND the verifier confirms** (`verified` if the
deterministic parser also agrees, else `high`); anything else is **flagged for human review and
never written**. Interactively the agent signals run on the operator's Claude subscription (no API
key); the standalone `claude_extract` batch path is the only one needing `ANTHROPIC_API_KEY`. The
line-based deterministic parser fails on several of these formats (HTML one-cell-per-line; broker
Schedule-I; multi-segment) — by design the two extract agents then carry the extraction, with the
adversarial verifier as the guard against the wrong-statement trap.

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
