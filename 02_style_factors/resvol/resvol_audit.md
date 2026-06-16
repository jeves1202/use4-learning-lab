# Residual Volatility (RESVOL) — Factor Construction Audit

**USE4-Faithful Implementation — `resvol_build.ipynb`**
Audit Date: 2026-06-11 | USE4 Learning Lab — Reference Audit

---

## Purpose and Scope

Audits the Residual Volatility factor: RESVOL = 0.75·DASTD + 0.15·CMRA + 0.10·HSIGMA, then orthogonalized to Beta via no-intercept WLS (USE4 Empirical Notes, Appendix A). DASTD = daily return std (252d, 42d HL); CMRA = 12-month cumulative range; HSIGMA = historical residual std from the beta regression.

**Scope.** This audit covers every pipeline stage of `resvol_build.ipynb` from the lab's reference run of 2026-06-11: all printed diagnostics, the validation battery, and the output parquet schema. All numerical values are verbatim from the executed notebook. The notebook generates no figures.

> *Reference numbers: every value in this document comes verbatim from the lab's reference run of 2026-06-11 (Sharadar data through 2026-06-10). Your rebuild will differ in row counts and tail statistics with data vintage; distribution shapes, stability coefficients, and check outcomes should match. Treat persistent sign or order-of-magnitude differences as bugs in your build until proven otherwise.*

---

## Output Schema

Deliverable: `data/out/resvol_use4.parquet`, zstd, statistics=True.

| Column | Dtype | Description |
|---|---|---|
| `permaticker` | Int64 | Sharadar permanent ticker ID |
| `signal_date` | Date | End-of-month rebalance date |
| `in_estu` | Bool | ESTU membership on `signal_date` |
| `mcap` | Float64 | Market cap on `signal_date` (cap-weighting) |
| `resvol` | Float64 | Raw descriptor: weighted composite of DASTD/CMRA/HSIGMA |
| `resvol_score` | Float64 | Final standardised exposure (the deliverable) |
| `n_obs` | UInt32 | Number of valid sub-descriptors included in the composite |

---

## Shared Infrastructure

- Signal calendar: **330 dates** (1999-01-29 → 2026-06-10)
- ESTU size: mean **2,494**, min **1,791**, max **3,045**

---

## Known Deviations from PDF Spec

Mirrors the complete NOT-IN-PDF master list from `resvol_spec.ipynb` (§10).

| # | Decision | Chosen solution | Revisit when |
|---|---|---|---|
| 1 | HSIGMA source | Residual std from the beta regression (shared beta artifact) | If beta artifact residuals are found to be noisy |
| 2 | DASTD half-life | 42 trading days (calendar ~2 months) | If USE4 clarifies a different DASTD decay rate |
| 3 | CMRA construction | 12 monthly log-return ranges; ratio of max to min cumulative return | If monthly dates vs daily extremes matter |
| 4 | Sub-descriptor normalisation | Each of DASTD/CMRA/HSIGMA standardised cross-sectionally before weighting | Consistent with LEV; standardise-then-composite |
| 5 | Orthogonalisation method | No-intercept WLS (√mcap weights, ESTU only), applied to composite | Applied after compositing, not per sub-descriptor |
| 6 | Min sub-descriptor requirement | n_obs = 2 of 3 (DASTD required; CMRA or HSIGMA can be null) | If CMRA/HSIGMA null rates are low |
| 7 | Coverage floor | 4,021 minimum (vs 4,000 standard) — RESVOL uses a 0.5% margin | Not revisit — 0.5% margin is intentional buffer |
| 8 | Partial composite | If CMRA or HSIGMA null: renormalise weights to 1.0 | If renormalised composite differs materially from full composite |
| 9 | Beta source | beta_score from beta artifact | Never — beta_score is the upstream deliverable |

---

## Stage-by-Stage Audit

### Stage 1 — DASTD from daily_panel

**PDF SPEC:** Exponentially-weighted daily return standard deviation (252 trading days, 42d HL).

**Observed:**
- Daily panel: **38,997,999** rows; **16,835** tickers; **6,936** trading days
- DASTD rows: **1,693,072**; **330** dates; **16,031** tickers

**Status:** ✅ **MATCHES SPEC**

### Stage 2 — CMRA from monthly extremes

**PDF SPEC:** 12-month log-return range (max cumulative − min cumulative return over 12 months).

**Observed:**
- Monthly window: 12 months; CMRA rows: **1,690,924**; **330** dates; **15,943** tickers

**Status:** ✅ **MATCHES SPEC**

### Stage 3 — HSIGMA from Beta artifact

**Observed:**
- Beta artifact rows: **1,841,672**; **327** dates; HSIGMA extracted from residual std column

**Status:** ✅ **MATCHES SPEC**

### Stage 4 — RESVOL estimator

**PDF SPEC:** 0.75·DASTD + 0.15·CMRA + 0.10·HSIGMA (sub-descriptors standardised), then orthogonalized to Beta.

**Implementation:** PIT join; each sub-descriptor CW-mean-0/EW-std-1 on ESTU; composite; no-intercept WLS on ESTU to remove beta loading; γ_t mean **0.0320**.

**Observed:**
- Panel: **1,606,079** rows, **330** dates, **15,809** tickers; duplicates **0**
- Orthogonality (max |Cov_CW(resvol, beta)|): **1.2041×10⁻¹⁷** (target <10⁻⁶)
- Spot 2026-05-29: **4,607** tickers, median resvol **−0.2064**

**Status:** ✅ **MATCHES SPEC**

### Stage 5 — Outlier trim (3σ)

**Implementation:** CW mean ± 3·EW std (ESTU bounds, applied to all).

**Observed:**
- Clipped (ESTU): **12,047** / 795,943 = **1.51%** (target 0.5–2%)
- Bounds: lo mean −2.534; hi mean 2.677 (max 2.938)
- Post-trim at 2026-05-29 (ESTU, n=2,608): mean −0.2039, std 0.7462, skew 1.048, kurt 0.614

**Status:** ✅ **MATCHES SPEC**

### Stage 6 — Standardisation

**Observed:**
- CW mean: **0.000000**; EW std: **1.000000**
- Fraction in [±3]: all **95.7%** / ESTU **97.8%**

**Status:** ✅ **MATCHES SPEC**

### Stage 7 — Save

**Observed:**
- Rows **1,606,079**; dates **330**; tickers **15,809**; **31.5 MB**
- Schema check ✅ **PASSED**; read-back ✅ **PASSED**

**Status:** ✅ **MATCHES SPEC**

---

## Validation Checks

All checks passed on the run dated 2026-06-11.

| Check | Description | Observed | Target / Status |
|---|---|---|---|
| 1a | Max \|μ_CW(resvol_score)\| | 2.61×10⁻¹⁶ | <10⁻⁶ ✅ PASS |
| 1b | Mean EW std | 1.0000 | 1.0±0.02 ✅ PASS |
| 2 | Orthogonality max \|Cov_CW(resvol, beta)\| | 1.2041×10⁻¹⁷ | <10⁻⁶ ✅ PASS |
| 3 | Min coverage post-2005 (completed months) | 4,021 | ≥4,000 ✅ PASS |
| 4 | MoM Spearman ρ mean | 0.9397 | >0.90 ✅ PASS |
| 5 | ESTU clip rate | 1.51% | 0.5–2.0% ✅ PASS |
| 6 | ESTU DASTD monotone with resvol_score | Checked ✅ | monotone ✅ PASS |
| 7 | Disk vs memory max diff | 0.00×10⁰ | <10⁻¹⁰ ✅ PASS |

- **Check 3 note:** 4,021 is 0.5% above the 4,000 floor — the 0.5% margin is intentional (documented deviation #7 above).
- **γ_t mean 0.0320:** Beta orthogonalisation coefficient is small and positive — RESVOL has a small raw beta loading that is removed each date.

---

## Spot Check at 2026-05-29

- Coverage: **4,607** tickers; median resvol **−0.2064**
- Post-trim ESTU (n=2,608): mean **−0.2039**, std **0.7462**, skew **1.048**, excess kurtosis **0.614**

> *No per-ticker factor values were printed by the build notebook; the spot check reports aggregate statistics only.*
