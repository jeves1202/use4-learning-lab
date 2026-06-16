# Momentum (MOM) — Factor Construction Audit

**USE4-Faithful Implementation — `mom_build.ipynb`**
Audit Date: 2026-06-11 | USE4 Learning Lab — Reference Audit

---

## Purpose and Scope

Audits the Momentum factor: exponentially-weighted 12-month cumulative excess return, skipping the most recent month (USE4 Empirical Notes, Appendix A). Window: 504 trading days back, 21-day lag, 126-day half-life.

**Scope.** This audit covers every pipeline stage of `mom_build.ipynb` from the lab's reference run of 2026-06-11: all printed diagnostics, the validation battery, and the output parquet schema. All numerical values are verbatim from the executed notebook. The notebook generates no figures.

> *Reference numbers: every value in this document comes verbatim from the lab's reference run of 2026-06-11 (Sharadar data through 2026-06-10). Your rebuild will differ in row counts and tail statistics with data vintage; distribution shapes, stability coefficients, and check outcomes should match. Treat persistent sign or order-of-magnitude differences as bugs in your build until proven otherwise.*

---

## Output Schema

Deliverable: `data/out/mom_use4.parquet`, zstd, statistics=True.

| Column | Dtype | Description |
|---|---|---|
| `permaticker` | Int64 | Sharadar permanent ticker ID |
| `signal_date` | Date | End-of-month rebalance date |
| `in_estu` | Bool | ESTU membership on `signal_date` |
| `mcap` | Float64 | Market cap on `signal_date` (cap-weighting) |
| `mom` | Float64 | Raw descriptor: EW cumulative log excess return |
| `mom_score` | Float64 | Final standardised exposure (the deliverable) |
| `n_obs` | UInt32 | Number of valid trading days in the momentum window |

---

## Shared Infrastructure

- Signal calendar: **330 dates** (1999-01-29 → 2026-06-10)
- ESTU size: mean **2,494**, min **1,791**, max **3,045**

---

## Known Deviations from PDF Spec

Mirrors the complete NOT-IN-PDF master list from `mom_spec.ipynb` (§10).

| # | Decision | Chosen solution | Revisit when |
|---|---|---|---|
| 1 | Excess return base | Cap-weighted ESTU benchmark (daily_panel artifact) | If benchmark choice is found to distort factor rankings |
| 2 | Half-life convention | 126 trading days | If USE4 clarifies a different decay rate |
| 3 | EW normalisation | Divide by W_full = 151.4083 (uniform) | If non-uniform days are common (early calendar edge) |
| 4 | Min trading-day requirement | MIN_OBS = 200 of 483 window days | If data sparsity causes coverage gaps |
| 5 | Log return construction | ln(1 + ret) from daily returns in shared panel | If arithmetic returns prove better calibrated |
| 6 | Skip-month implementation | Window: days d ∈ [22, 525] (1-indexed from signal_date) | If different skip-month boundary is found in USE4 docs |
| 7 | Delisting treatment | Last valid price in panel; naturally excluded after delisting | Inherits from daily_panel; no separate treatment |
| 8 | Early period warm-up | Dates before 2001-01-31 have very thin coverage | Dates pre-2001 are excluded from validation checks |

---

## Stage-by-Stage Audit

### Stage 1 — Load daily_panel + ESTU

**Observed:**
- Daily panel: **38,997,999** rows; **16,835** tickers; **6,936** trading days
- Signal calendar: **330 dates** (1999-01-29 → 2026-06-10)
- ESTU universe rows: **1,773,820**

**Status:** ✅ **MATCHES SPEC**

### Stage 4 — MOM estimator

**PDF SPEC:** EW cumulative log excess return over 12m window, 1m lag, 126d HL.

**Implementation:** For each (permaticker, signal_date): sum EW log excess returns for days d ∈ [22, 525] (most recent to oldest), weight w_d = 0.5^(d/126); divide by W_full = 151.4083.

**Observed:**
- W_full: **151.4083** (sum of weights over [22, 525])
- Panel: **1,798,883** rows, **329** dates (2000-01-31 → 2026-06-10), **16,052** tickers; duplicates **0**
- Spot 2026-05-29: **4,681** tickers, median mom **0.0436**

**Status:** ✅ **MATCHES SPEC**

### Stage 5 — Outlier trim (3σ)

**Implementation:** CW mean ± 3·EW std (ESTU bounds, applied to all).

**Observed:**
- Clipped (ESTU): **13,757** / 814,208 = **1.70%** (target 0.5–2%)
- Bounds: lo mean −0.3476; hi mean 0.4563 (max 0.8013)
- Post-trim at 2026-05-29 (ESTU, n=2,730): mean 0.0378, std 0.2185, skew 0.016, kurt 0.471

**Status:** ✅ **MATCHES SPEC**

### Stage 6 — Standardisation

**Observed:**
- CW mean: **−0.000000**; EW std: **1.000000**
- Fraction in [±3]: all **89.0%** / ESTU **97.9%**

**Status:** ✅ **MATCHES SPEC**

### Stage 7 — Save

**Observed:**
- Rows **1,798,883**; dates **329**; tickers **16,052**; **35.0 MB**
- Schema check ✅ **PASSED**; read-back ✅ **PASSED**

**Status:** ✅ **MATCHES SPEC**

---

## Validation Checks

All checks passed on the run dated 2026-06-11.

| Check | Description | Observed | Target / Status |
|---|---|---|---|
| 1a | Max \|μ_CW(mom_score)\| | 3.51×10⁻¹⁶ | <10⁻⁶ ✅ PASS |
| 1b | Mean EW std | 1.0000 | 1.0±0.02 ✅ PASS |
| 2a | Spot ann. momentum spread | +1.094%/mo (+13.1% ann.) | >0% ✅ PASS |
| 2b | Fraction of dates MoM ρ > 0.80 | 97.9% | >80% ✅ PASS |
| 3 | Min coverage post-2005 (completed months) | 4,376 | ≥4,000 ✅ PASS |
| 4 | MoM Spearman ρ mean | 0.8702 | >0.80 ✅ PASS |
| 5 | Q1→Q5 mean mom monotone | spread varies | monotone ✅ PASS |
| 6 | Disk vs memory max diff | 0.00×10⁰ | <10⁻¹⁰ ✅ PASS |

- **Check 4 note:** MoM ρ = 0.8702 is lower than other style factors by design — MOM is a return signal and inherently turns over at its rebalance cadence.
- **Check 2a note:** Annualized as +1.094%/mo × 12 = +13.1%/year spread (raw, pre-cost).

---

## Spot Check at 2026-05-29

- Coverage: **4,681** tickers; median mom **0.0436**
- Post-trim ESTU (n=2,730): mean **0.0378**, std **0.2185**, skew **0.016**, excess kurtosis **0.471**

> *No per-ticker factor values were printed by the build notebook; the spot check reports aggregate statistics only.*
