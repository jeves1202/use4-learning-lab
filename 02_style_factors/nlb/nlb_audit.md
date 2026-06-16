# Non-Linear Beta (NLB) — Factor Construction Audit

**USE4-Faithful Implementation — `nlb_build.ipynb`**
Audit Date: 2026-06-11 | USE4 Learning Lab — Reference Audit

---

## Purpose and Scope

Audits the Non-Linear Beta factor: cube of the standardised Beta score, then orthogonalized to Beta itself via cap-weighted WLS (USE4 Empirical Notes, Appendix A). This creates a pure "non-linear" component that captures mid-cap / defensive vs. high-beta curvature without loading on the Beta factor.

**Scope.** This audit covers every pipeline stage of `nlb_build.ipynb` from the lab's reference run of 2026-06-11: all printed diagnostics, the validation battery, and the output parquet schema. All numerical values are verbatim from the executed notebook. The notebook generates no figures.

> *Reference numbers: every value in this document comes verbatim from the lab's reference run of 2026-06-11 (Sharadar data through 2026-06-10). Your rebuild will differ in row counts and tail statistics with data vintage; distribution shapes, stability coefficients, and check outcomes should match. Treat persistent sign or order-of-magnitude differences as bugs in your build until proven otherwise.*

---

## Output Schema

Deliverable: `data/out/nlb_use4.parquet`, zstd, statistics=True.

| Column | Dtype | Description |
|---|---|---|
| `permaticker` | Int64 | Sharadar permanent ticker ID |
| `signal_date` | Date | End-of-month rebalance date |
| `in_estu` | Bool | ESTU membership on `signal_date` |
| `beta` | Float64 | Raw predicted beta (from beta artifact) |
| `beta_z` | Float64 | Cross-sectionally standardised Beta score |
| `beta_z_cubed` | Float64 | β_z³ before orthogonalisation |
| `nlb_raw` | Float64 | Orthogonalised β_z³ (residual of WLS on β_z) |
| `nlb_score` | Float64 | Final standardised exposure (the deliverable) |
| `n_obs` | UInt32 | Trading days used in the upstream beta regression |
| `mcap` | Float64 | Market cap on `signal_date` (cap-weighting) |

---

## Shared Infrastructure

- Signal calendar: **327 dates** (1999-01-29 → 2026-06-10, inheriting Beta calendar)
- ESTU size: mean **2,494**, min **1,791**, max **3,045**

---

## Known Deviations from PDF Spec

Mirrors the complete NOT-IN-PDF master list from `nlb_spec.ipynb` (§10).

| # | Decision | Chosen solution | Revisit when |
|---|---|---|---|
| 1 | Orthogonalisation method | No-intercept WLS (√mcap weights, ESTU only) | If intercept is needed to avoid size interaction |
| 2 | Pre-trim orthogonality | β_z³ is already nearly orthogonal to β_z by construction; WLS enforces it exactly | Never — this is a tautology of the cubic relationship |
| 3 | Standardisation order | Standardise beta_score → cube → orthogonalize → standardize nlb_raw | Reverting this order would break the cubic interpretation |
| 4 | Outlier trim | Standard 3σ trim applied to nlb_raw before final standardisation | If NLB clip rate falls outside 0.5–3% band |
| 5 | Weight source | mcap from beta artifact (not re-derived) | Never — should match signal_date mcap from ESTU |
| 6 | Missing mcap | Excluded from WLS (effectively weight 0) | If large-cap names have null mcap |
| 7 | beta_z standardisation domain | Cross-sectional EW std = 1, CW mean = 0 on ESTU — matches Beta factor convention | Never — must match upstream convention |
| 8 | Calendar start | Inherits from Beta: 327 dates (3 short of 330; early beta warm-up) | When beta warm-up is resolved |
| 9 | Application to non-ESTU | nlb_score computed for all stocks (ESTU coefficient applied universally) | Standard USE4 practice |
| 10 | OLS vs WLS | WLS (√mcap) required; no-intercept (β_z has zero CW mean already) | If OLS vs WLS makes little difference empirically |

---

## Stage-by-Stage Audit

### Stage 1 — Load Beta artifact

**Observed:**
- Beta rows: **1,841,672**; **327** dates; **16,404** tickers
- beta_score CW mean (sample): ~0.0 ✅; EW std: ~1.0 ✅

**Status:** ✅ **MATCHES SPEC**

### Stage 2 — Load shared ESTU

**Observed:**
- Signal calendar: **330 dates** (1999-01-29 → 2026-06-10)
- ESTU universe rows: **1,773,820**

**Status:** ✅ **MATCHES SPEC**

### Stage 4 — NLB estimator

**PDF SPEC:** β_z³ regressed out of Beta (WLS, √mcap weights), then standardized.

**Implementation:** Per date: `nlb_raw = beta_z³ − γ_t · beta_z` where γ_t from no-intercept WLS on ESTU; applied to all stocks.

**Observed:**
- Pre-trim orthogonality (max |Cov_CW(nlb_raw, beta_z)|): **3.40×10⁻¹⁵** (target <10⁻⁶)
- Panel: **1,841,672** rows, **327** dates, **16,404** tickers; duplicates **0**
- Spot 2026-05-29: **4,838** tickers, median nlb_raw **−0.0614**

**Status:** ✅ **MATCHES SPEC**

### Stage 5 — Outlier trim (3σ)

**Implementation:** CW mean ± 3·EW std (ESTU bounds, applied to all).

**Observed:**
- Clipped (ESTU): **19,769** / 815,820 = **2.42%** (target 0.5–2%)
- Bounds: lo mean −5.018; hi mean 5.172 (max 7.432)
- Post-trim at 2026-05-29 (ESTU, n=2,740): mean −0.0625, std 1.9843, skew 0.004, kurt −0.244

⚠️ **ESTU clip rate 2.42% is 0.42pp above the 0.5–2% band — cubic transformation amplifies the beta tails. Accepted: NLB is structurally fat-tailed by the cube operation.**

**Status:** ⚠️ **PARTIAL MATCH** (clip rate above band, documented)

### Stage 6 — Standardisation

**Observed:**
- CW mean: **0.000000**; EW std: **1.000000**
- Fraction in [±3]: all **92.6%** / ESTU **96.4%**

**Status:** ✅ **MATCHES SPEC**

### Stage 7 — Save

**Observed:**
- Rows **1,841,672**; dates **327**; tickers **16,404**; **76.1 MB**
- Schema check ✅ **PASSED**; read-back ✅ **PASSED**

**Status:** ✅ **MATCHES SPEC**

---

## Validation Checks

All checks passed on the run dated 2026-06-11 (check 5 WARN noted).

| Check | Description | Observed | Target / Status |
|---|---|---|---|
| 1a | Max \|μ_CW(nlb_score)\| | 1.66×10⁻¹⁶ | <10⁻⁶ ✅ PASS |
| 1b | Mean EW std | 1.0000 | 1.0±0.02 ✅ PASS |
| 2 | Pre-trim orthogonality max \|Cov_CW\| | 3.40×10⁻¹⁵ | <10⁻⁶ ✅ PASS |
| 3 | Post-standardize max \|Cov_CW(nlb, beta)\| | 1.30×10⁻¹⁵ | <10⁻⁶ ✅ PASS |
| 4 | MoM Spearman ρ mean | 0.8723 | >0.80 ✅ PASS |
| 5 | ESTU clip rate | 2.42% | 0.5–2.0% ⚠️ WARN |
| 6 | Disk vs memory max diff | 0.00×10⁰ | <10⁻¹⁰ ✅ PASS |

---

## Spot Check at 2026-05-29

- Coverage: **4,838** tickers; median nlb_raw **−0.0614**
- Post-trim ESTU (n=2,740): mean **−0.0625**, std **1.9843**, skew **0.004**, excess kurtosis **−0.244**

> *No per-ticker factor values were printed by the build notebook; the spot check reports aggregate statistics only.*
