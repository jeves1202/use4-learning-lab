# Beta (BETA) — Factor Construction Audit

**USE4-Faithful Implementation — `beta_build.ipynb`**
Audit Date: 2026-06-11 | USE4 Learning Lab — Reference Audit

---

## Purpose and Scope

Audits the Beta factor: exponentially-weighted OLS slope of stock excess returns on the ESTU cap-weighted excess benchmark over a trailing 252-day window with a 63-day half-life (USE4 Empirical Notes, Appendix A). `alpha` and `r2` are retained for audit.

**Scope.** This audit covers every pipeline stage of `beta_build.ipynb` from the lab's reference run of 2026-06-11: all printed diagnostics, the validation battery, and the output parquet schema. All numerical values are verbatim from the executed notebook. The notebook generates no figures.

> *Reference numbers: every value in this document comes verbatim from the lab's reference run of 2026-06-11 (Sharadar data through 2026-06-10). Your rebuild will differ in row counts and tail statistics with data vintage; distribution shapes, stability coefficients, and check outcomes should match. Treat persistent sign or order-of-magnitude differences as bugs in your build until proven otherwise.*

---

## Output Schema

Deliverable: `data/out/beta_use4.parquet`, zstd, statistics=True.

| Column | Dtype | Description |
|---|---|---|
| `permaticker` | Int64 | Sharadar permanent ticker ID |
| `signal_date` | Date | End-of-month rebalance date |
| `in_estu` | Bool | ESTU membership |
| `mcap` | Float64 | Market cap on signal date |
| `beta` | Float64 | Raw descriptor: EW-OLS slope, post-trim |
| `alpha` | Float64 | Regression intercept (audit column) |
| `beta_score` | Float64 | Final standardised exposure |
| `n_obs` | UInt32 | Valid observations in the window (≥ 63) |
| `r2` | Float64 | Regression R² (audit column) |

---

## Shared Infrastructure

- Signal calendar: **330 dates** (1999-01-29 → 2026-06-10)
- ESTU size: mean **2,494**, min **1,791**, max **3,045**
- Daily artifact: **38,997,999** rows, **17,413** tickers (1998-01-02 → 2026-06-10); loader mode **FAST** (commit-aware gate)
- `_td_to_sig`: **7,153** trading days mapped; missing-benchmark dates **0** (target < 30)

---

## Known Deviations from PDF Spec

Mirrors the complete NOT-IN-PDF master list from `beta_spec.ipynb` (§13).

| # | Decision | Chosen solution | Revisit when |
|---|---|---|---|
| 1 | ESTU construction (MSCI USA IMI is proprietary) | Shared `estu_monthly`: top 3000, ATVR screen, 3000/3500 buffer | If ESTU churn > 5%/month |
| 2 | Risk-free rate for excess returns | Ken French daily RF, subtracted in the shared daily artifact | If validating against USE4 published betas |
| 3 | Min observations for beta | 63 (~3 months) | If coverage too low at start of sample |
| 4 | Outlier treatment | 3-sigma trim per Section 2.2 (optional) | If validation shows extreme beta values dominating cross-section |
| 5 | ESTU buffer rule for stability | Inherited from shared ESTU (3000/3500 hysteresis) | If month-over-month stability < 0.94 |
| 6 | Daily vs monthly ESTU for benchmark | Monthly (locked at signal_date) | Probably never |
| 7 | Float-adjusted mcap ⚠️ **Material** | Total mcap (Sharadar doesn't have float) | If running for institutional-grade comparison to MSCI |
| 8 | Calendar start | 1999-01-01 | If we want full sample |

---

## Stage-by-Stage Audit

### Stage 1 — Shared daily-returns artifact

**PDF SPEC:** Excess returns and the ESTU cap-weighted benchmark.

**Observed:**
- **38,997,999** rows, **17,413** tickers (1998-01-02 → 2026-06-10)
- Working set ≈1.4 GB; mode **FAST**

**Status:** ✅ **MATCHES SPEC**

### Stage 2 — Load shared ESTU

**Observed:**
- 330 dates (1999-01-29 → 2026-06-10); in-ESTU rows **822,918**; size mean **2,494**
- Spot: JPM/MSFT **330** months; GOOGL **261**; AAPL **257**

**Status:** ✅ **MATCHES SPEC**

### Stage 4 — Per-stock Beta (EW-OLS)

**PDF SPEC:** 252-day window, 63-day half-life exponential weights, w_τ = exp(−ln2·(t−τ)/63).

**Observed:**
- Numpy-reference equivalence at 2026-05-29: max diff **2.66×10⁻¹⁵** (target <10⁻¹⁰); median R² 0.075
- Raw panel: **1,841,672** rows, **327** dates, **16,404** tickers; duplicates **0**
- Cross-sectional median beta **0.852**; CW ESTU beta **1.007** (target ≈1.0); betas in [0, 2.5]: **83.5%**

**Status:** ✅ **MATCHES SPEC**

### Stage 5 — Outlier trim (3σ)

**PDF SPEC:** Methodology §2.2 trim.

**Implementation:** CW mean ± 3·EW std (ESTU bounds, applied to all).

**Observed:**
- Clipped (ESTU): **10,024** / 813,270 = **1.23%** (target 0.5–2%)
- Bounds: lo mean −0.567; hi mean 2.580 (max 3.863)
- Post-trim at 2026-05-29 (ESTU, n=2,746): mean 1.134, std 0.836, skew 0.742, kurt 0.700

**Status:** ✅ **MATCHES SPEC**

### Stage 6 — Standardisation

**PDF SPEC:** CW mean 0 (ESTU), EW std 1 (ESTU), applied to all stocks.

**Observed:**
- CW mean: **−0.000000**; EW std: **1.000000**
- Fraction in [±3] at 2026-05-29: **97.4%**
- Spot scores: GOOGL +0.352, MSFT −0.270, AAPL −0.294, JPM −0.169, NEE −1.133, KO −1.404

**Status:** ✅ **MATCHES SPEC**

### Stage 7 — Save

**Observed:**
- Rows **1,841,672**; dates **327** (1999-04-30 → 2026-06-10); tickers **16,404**; **63.4 MB**
- Schema check ✅ **PASSED**; read-back ✅ **PASSED**

**Status:** ✅ **MATCHES SPEC**

---

## Validation Checks

All checks passed on the run dated 2026-06-11. Coverage (check 3) is evaluated on completed months only.

| Check | Description | Observed | Target / Status |
|---|---|---|---|
| 1a | Max \|μ_CW(beta_score)\| | 8.39×10⁻¹⁶ | <10⁻⁶ ✅ PASS |
| 1b | Mean EW std | 1.0000 | 1.0±0.02 ✅ PASS |
| 2 | Max \|μ_CW(raw beta)−1\| | 0.1629 | <0.20 ✅ PASS |
| 3 | Min coverage post-2005 (completed months) | 4,854 | ≥4,000 ✅ PASS |
| 4 | MoM Spearman ρ mean | 0.9458 | >0.94 ✅ PASS |
| 5 | Q5−Q1 vs ESTU mkt corr | 0.7475 | >0.70 ✅ PASS |
| 6 | Disk vs memory max diff | 0.00×10⁰ | <10⁻¹⁰ ✅ PASS |

- **Check 2:** worst-date deviation 0.1629 at 2001-02-28 (ESTU n=2,081); mean deviation 0.0180.
- **Check 4:** ρ > 0.94 on 71.8% of months; > 0.90 on 87.1% (USE4 Table 4.2 reports 0.96–0.97).

---

## Spot Check at 2026-05-29

- **5,204** tickers matched in the equivalence check; alpha/r2 non-null for all
- Post-trim ESTU (n=2,746): mean **1.134**, std **0.836**, skew **0.742**, excess kurtosis **0.700**

> *No per-ticker factor values were printed by the build notebook beyond the six named tickers in Stage 6; the spot check reports aggregate statistics only.*
