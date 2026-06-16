# Earnings Yield (EYLD) — Factor Construction Audit

**USE4-Faithful Implementation — `eyld_build.ipynb`**
Audit Date: 2026-06-11 | USE4 Learning Lab — Reference Audit

---

## Purpose and Scope

Audits the Earnings Yield factor: EYLD = 0.6·CETOP + 0.4·ETOP (USE4 Empirical Notes, Appendix A), with ETOP = TTM netinc / mcap and CETOP = (TTM netinc + TTM depamor) / mcap as the cash-earnings proxy.

**Scope.** This audit covers every pipeline stage of `eyld_build.ipynb` from the lab's reference run of 2026-06-11: all printed diagnostics, the validation battery, and the output parquet schema. All numerical values are verbatim from the executed notebook. The notebook generates no figures.

> *Reference numbers: every value in this document comes verbatim from the lab's reference run of 2026-06-11 (Sharadar data through 2026-06-10). Your rebuild will differ in row counts and tail statistics with data vintage; distribution shapes, stability coefficients, and check outcomes should match. Treat persistent sign or order-of-magnitude differences as bugs in your build until proven otherwise.*

---

## Output Schema

Deliverable: `data/out/eyld_use4.parquet`, zstd, statistics=True.

| Column | Dtype | Description |
|---|---|---|
| `permaticker` | Int64 | Sharadar permanent ticker ID |
| `signal_date` | Date | End-of-month rebalance date |
| `in_estu` | Bool | ESTU membership on `signal_date` |
| `mcap` | Float64 | Market cap on `signal_date` (cap-weighting) |
| `eyld` | Float64 | Raw descriptor: 0.6·CETOP + 0.4·ETOP |
| `eyld_score` | Float64 | Final standardised exposure (the deliverable) |
| `n_obs` | UInt32 | 1 if a valid TTM observation within the lookback window |

---

## Shared Infrastructure

- Signal calendar: **330 dates** (1999-01-29 → 2026-06-10)
- ESTU size: mean **2,494**, min **1,791**, max **3,045**

---

## Known Deviations from PDF Spec

Mirrors the complete NOT-IN-PDF master list from `eyld_spec.ipynb` (§13).

| # | Decision | Chosen solution | Revisit when |
|---|---|---|---|
| 1 | EPFWD omitted — no forward estimates in database ⚠️ **Material** | Renormalize to 0.6·CETOP + 0.4·ETOP | If forward estimate data becomes available |
| 2 | Data source for fundamentals | Sharadar SF1 ARQ; TTM constructed as 4-quarter rolling sums | Never (Sharadar is the lab's data stack) |
| 3 | PIT alignment | Use `datekey` (filing date) as PIT constraint | If Sharadar datekey quality is suspected |
| 4 | Cash earnings definition | netinc + depamor | If different definition matches USE4 intent better |
| 5 | mcap denominator source | estu_monthly mcap on signal_date | Never — signal_date mcap is always correct for E/P ratios |
| 6 | Negative earnings treatment | Allow; 3σ trim handles extremes | If factor is used in a value-only context |
| 7 | Staleness cutoff | 548 calendar days (~18 months) | If stale fundamentals are suspected to add noise |
| 8 | Financial sector treatment | No special treatment; include all sectors | If financial sector distorts the factor |
| 9 | MIN_OBS threshold | 1 (binary: has valid TTM or not) | If n_obs proves uninformative |
| 10 | Calendar start | 1999-01-01 | If early data quality is poor |

---

## Stage-by-Stage Audit

### Stage 1 — SF1 → TTM earnings

**PDF SPEC:** Trailing 12-month earnings and cash earnings.

**Observed:**
- TTM rows: **567,298**; **15,250** tickers (1999-01-04 → 2026-06-10)
- netinc nulls **0**; depamor nulls **0**
- Two polars DeprecationWarnings (`min_periods` → `min_samples`) — cosmetic, behaviour unchanged

**Status:** ✅ **MATCHES SPEC**

### Stage 2 — Load shared ESTU

**Observed:**
- Signal calendar: **330 dates** (1999-01-29 → 2026-06-10)
- ESTU universe rows: **1,773,820**
- ESTU size: mean **2,494**, min **1,791**, max **3,045**

**Status:** ✅ **MATCHES SPEC**

### Stage 4 — EYLD estimator (PIT join)

**Implementation:** `join_asof` backward on `datekey`; staleness ≤ 548 days; `eyld` = 0.6·(netinc+depamor)/mcap + 0.4·netinc/mcap.

**Observed:**
- Rows with SF1 hit: **1,548,088**; panel **1,537,393** rows, **14,933** tickers; duplicates **0**
- Spot 2026-05-29: **4,210** tickers, median eyld **0.0332**

**Status:** ✅ **MATCHES SPEC**

### Stage 5 — Outlier trim (3σ)

**Implementation:** CW mean ± 3·EW std (ESTU bounds, applied to all).

**Observed:**
- Clipped (ESTU): **10,216** / 791,992 = **1.29%** (target 0.5–2%)
- Bounds: lo mean −0.5693; hi mean 0.6875
- Post-trim at 2026-05-29 (ESTU, n=2,708): mean 0.0266, std 0.1303, skew −1.388, kurt 7.885

**Status:** ✅ **MATCHES SPEC**

### Stage 6 — Standardisation

**Observed:**
- CW mean: **−0.000000**; EW std: **1.000000**
- Fraction in [±3]: all **87.5%** / ESTU **97.2%**

**Status:** ✅ **MATCHES SPEC**

### Stage 7 — Save

**Observed:**
- Rows **1,537,393**; dates **330**; tickers **14,933**; **26.5 MB**
- Schema check ✅ **PASSED**; read-back ✅ **PASSED**

**Status:** ✅ **MATCHES SPEC**

---

## Validation Checks

All checks passed on the run dated 2026-06-11.

| Check | Description | Observed | Target / Status |
|---|---|---|---|
| 1a | Max \|μ_CW(eyld_score)\| | 1.90×10⁻¹⁶ | <10⁻⁶ ✅ PASS |
| 1b | Mean EW std | 1.0000 | 1.0±0.02 ✅ PASS |
| 2a | Median raw eyld (ESTU) | 0.05891 | >0.0 ✅ PASS |
| 2b | Dates with positive ESTU median | 100.0% | >70% ✅ PASS |
| 3 | Min coverage post-2005 (completed months) | 4,151 | ≥4,000 ✅ PASS |
| 4 | MoM Spearman ρ mean | 0.9630 | >0.85 ✅ PASS |
| 5 | Q1→Q5 mean eyld monotone | spread 0.26033 | monotone ✅ PASS |
| 6 | Disk vs memory max diff | 0.00×10⁰ | <10⁻¹⁰ ✅ PASS |

- **Check 5 quintile means:** Q1 = −0.09620, Q2 = 0.03215, Q3 = 0.05821, Q4 = 0.08431, Q5 = 0.16414.

---

## Spot Check at 2026-05-29

- Coverage: **4,210** tickers; median eyld **0.0332**; ESTU **2,708** stocks
- Post-trim ESTU: mean **0.0266**, std **0.1303**, skew **−1.388**, excess kurtosis **7.885**

> *No per-ticker factor values were printed by the build notebook; the spot check reports aggregate statistics only.*
