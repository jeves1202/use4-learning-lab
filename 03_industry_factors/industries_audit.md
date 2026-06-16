# Industry Factors (INDUSTRIES) — Factor Construction Audit

**USE4-Faithful Implementation — `industries_build.ipynb`**
Audit Date: 2026-06-11 | USE4 Learning Lab — Reference Audit

---

## Purpose and Scope

Audits the industry-factor exposure layer: USE4 specifies 60 GICS-based industry factors with single 0/1 membership for most stocks (fractional for diversified firms via business-segment data). With GICS and segment data both absent from public Sharadar, the implementation delivers **55 factors** over a hand-engineered scheme (`industry_scheme.csv`: 154 Sharadar `industry` atoms → 55 factors + 1 fallback) with single membership throughout. Industries are dummies, not standardized descriptors — there is no trim, no standardisation, and the validation battery is membership-shaped.

**Scope.** This audit covers every pipeline stage of `industries_build.ipynb` as executed on 2026-06-11: all printed diagnostics, the 8-check validation battery, and the output parquet schema. All numerical values are verbatim from the executed notebook. The notebook generates no figures.

> *Reference numbers: every value in this document comes verbatim from the lab's reference run of 2026-06-11 (Sharadar data through 2026-06-10). Your rebuild will differ in row counts with data vintage. Your factor count will also differ from 55 if your scheme makes different merge/split decisions — that is expected and fine. The structural properties that should hold regardless: zero floor violations (or declared exceptions), 100% atom coverage, zero in-ESTU UNASSIGNED, static map integrity.*

---

## Output Schema

**Primary deliverable:** `data/out/industries_use4.parquet`, zstd, statistics=True.

| Column | Dtype | Description |
|---|---|---|
| `permaticker` | Int64 | Sharadar permanent ticker ID |
| `signal_date` | Date | End-of-month rebalance date |
| `in_estu` | Bool | ESTU membership on `signal_date` |
| `mcap` | Float64 | Market cap on `signal_date` |
| `sharadar_industry` | String | Raw classification atom (audit column) |
| `industry` | String | **The factor label** — one of 55 scheme factors |
| `use4_sector` | String | Sector grouping of the factor |

**Weights artifact:** `data/out/industry_weights_use4.parquet` — the per-date industry cap-weight vector (the w in USE4's identification constraint Σ_j w_j f_j^ind = 0):

| Column | Dtype | Description |
|---|---|---|
| `signal_date` | Date | End-of-month rebalance date |
| `industry` | String | Factor label |
| `n_members` | UInt32 | In-ESTU members on the date |
| `cap_weight` | Float64 | Industry share of in-ESTU cap (sums to 1 per date) |

---

## Shared Infrastructure

Consumes `data/out/estu_monthly.parquet`; independent of the daily-returns artifact. On this run:

- Signal calendar: **330 dates** (1999-01-29 → 2026-06-10)
- Universe rows: **1,773,820**; ESTU size mean **2,494**, min **1,791**, max **3,045**

---

## Known Deviations from PDF Spec

Mirrors the complete master list from `industries_spec.ipynb` (§9).

| # | Decision | Chosen solution | Revisit when |
|---|---|---|---|
| 1 | GICS unavailable ⚠️ **Material** | Sharadar `industry` atoms (154) + hand-engineered 55-factor scheme targeting the USE4 60-list | GICS or a crosswalk is licensed |
| 2 | No segment data ⚠️ **Material** | Single 0/1 membership for all stocks (USE4: ~63% of cap multi-exposed) | Segment-level data procured |
| 3 | Classification not PIT | Static current-snapshot labels | EDGAR 10-K SIC PIT layer if drift shown to matter |
| 4 | Floor rule | median ≥ 15 and min ≥ 8 ESTU members over the full sample | CSR explanatory-power screening replaces member counts |
| 5 | Floor exceptions (4) | Internet & Catalog Retail (AMZN); Managed Health Care (USE4-explicit); Airlines; Industrial Conglomerates | Any falls below ~5 members persistently |
| 6 | Merges vs the 60-list | Drilling→Equip&Svcs; Paper+Forest→Construction Materials; both metals→Metals & Mining; durables/apparel/leisure and retail consolidated; telecom 2→1 | CSR favors a split AND members support it |
| 7 | Splits vs the 60-list | Diversified Financials and Software each split in two (2026 tectonics: 107–151-member monsters as single factors) | Never — deliberate modernisation |
| 8 | Fat factors kept whole | Banks, Real Estate, Biotech & Life Sciences, Software–Application per USE4-2011 | CSR screening adjudicates |
| 9 | Judgment atoms | Solar→Semi Equipment; Consumer Electronics (incl. AAPL)→Computers & Electronics; SPACs→Consumer Finance & DFS | Reclassification evidence |
| 10 | REITs | One Real Estate factor under Financials (USE4-2011, pre-2016 GICS promotion) | Modern-GICS baseline adopted |

---

## Stage-by-Stage Audit

### Stage 1 — Scheme + classification load

**PDF SPEC:** GICS-based classification.

**Implementation:** `industry_scheme.csv` loaded and asserted against the observed atom universe — an unmapped atom fails the build (vendor taxonomy changes must be reviewed, never silently bucketed).

**Observed:**
- Scheme: **154** atoms → **55** factors (+ fallback)
- Tickers: **44,163** permatickers (deduplicated)
- Atom coverage: all 154 observed atoms mapped ✅ **PASS**

> *The fail-loudly assert fired on first execution: three atoms (Banks–Global, Drug Manufacturers–Major, Silver) exist in the full ticker universe but never appeared among ESTU members, so the ESTU-derived design worksheet missed them. They were reviewed and added to the scheme (Banks, Pharmaceuticals, Metals & Mining respectively) — the guard working exactly as specified.*

**Status:** ✅ **MATCHES SPEC**

### Stage 2 — Load shared ESTU

**PDF SPEC:** Exposures over the estimation universe.

**Implementation:** Shared `estu_monthly.parquet`, signal dates ≥ 1999-01-01.

**Observed:**
- **330 dates**; **1,773,820** universe rows; ESTU mean **2,494**

**Status:** ✅ **MATCHES SPEC**

### Stages 3–4 — Classification and panel

**PDF SPEC:** One primary industry exposure of 1.0 (fractional via segments for diversified firms).

**Implementation:** permaticker → atom → factor join; null atoms → `UNASSIGNED`; single membership throughout (segment data absent — headline deviation #2).

**Observed:**
- Panel: **1,773,820** rows, **330** dates, **16,829** tickers; duplicates **0**
- Max in-ESTU `UNASSIGNED` on any date: **1**
- Spot 2026-05-29: **2,761** ESTU members across **55** factors; largest: Biotechnology & Life Sciences **277**, Banks **225**, Real Estate **172**, HC Equipment & Technology **115**, Software–Application **107**

**Status:** ✅ **MATCHES SPEC**

### Stage 5 — Save

**Implementation:** zstd parquet; 7-column dtype assertion; read-back assertion.

**Observed:**
- Rows **1,773,820**; dates **330** (1999-01-29 → 2026-06-10); tickers **16,829**; **8.6 MB**
- Schema check ✅ **PASSED**; read-back ✅ **PASSED**
- Weights artifact: **18,150** rows (330 dates × 55 factors); per-date weight sums |Σw − 1| max **5.55×10⁻¹⁶**; largest single weight ever **14.1%**

**Status:** ✅ **MATCHES SPEC**

---

## Validation Checks

All 8 checks passed on the run dated 2026-06-11. The battery is membership-shaped (industries are dummies); the completed-months coverage convention does not apply because the map is static.

| Check | Description | Observed | Target / Status |
|---|---|---|---|
| 1 | In-ESTU `UNASSIGNED` per date | max 1 | ≤ 2 ✅ PASS |
| 2 | All 55 factors present every date | min 55, max 55 | 55 ✅ PASS |
| 3 | Floor (median ≥15, min ≥8, ex-exceptions) | 0 violations | 0 ✅ PASS |
| 4 | Static map (one label per permaticker) | 0 violators | 0 ✅ PASS |
| 5 | Single membership (no duplicate rows) | 0 dupes | 0 ✅ PASS |
| 6 | Largest factor cap share | 14.1% | ≤ 30% ✅ PASS |
| 7 | Ticker spot checks | all 6 correct | ✅ PASS |
| 8 | Disk vs memory | row-exact | ✅ PASS |

**Check 3 exceptions** (declared, kept on cap-weight / USE4-explicitness grounds): Airlines (med 12, min 9); Industrial Conglomerates (med 10, min 6); Internet & Catalog Retail (med 6, min 3); Managed Health Care (med 14, min 9).

**Check 7:** AAPL → Computers & Electronics; JPM → Banks; XOM → Oil, Gas & Consumable Fuels; AMZN → Internet & Catalog Retail; NVDA → Semiconductors; PLD → Real Estate.

---

## Spot Check at 2026-05-29

- **2,761** ESTU members classified across **55** factors
- Five largest factors by membership: Biotechnology & Life Sciences **277**, Banks **225**, Real Estate **172**, Health Care Equipment & Technology **115**, Software–Application **107**
- Largest factor cap share at any date in the sample: **14.1%**

> *Membership counts per factor are the spot statistics for a dummy layer; no per-ticker exposures exist beyond the six named spot checks above.*
