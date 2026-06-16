# Non-Linear Size (NLS) — Factor Construction Audit

**USE4-Faithful Implementation — `nls_build.ipynb`**
Audit Date: 2026-06-11 | USE4 Learning Lab — Reference Audit

---

## Purpose and Scope

Audits the Non-Linear Size factor: cube of the standardised Size score (log-mcap), then orthogonalized to Size via cap-weighted WLS. The cubing is applied after a 1%/99% winsorisation of the size score, then residualised against Size to produce a factor that is orthogonal by construction (USE4 Empirical Notes, Appendix A).

**Scope.** This audit covers every pipeline stage of `nls_build.ipynb` from the lab's reference run of 2026-06-11: all printed diagnostics, the validation battery, and the output parquet schema. All numerical values are verbatim from the executed notebook. The notebook generates no figures.

> *Reference numbers: every value in this document comes verbatim from the lab's reference run of 2026-06-11 (Sharadar data through 2026-06-10). Your rebuild will differ in row counts and tail statistics with data vintage; distribution shapes, stability coefficients, and check outcomes should match. Treat persistent sign or order-of-magnitude differences as bugs in your build until proven otherwise.*

---

## Output Schema

Deliverable: `data/out/nls_use4.parquet`, zstd, statistics=True.

| Column | Dtype | Description |
|---|---|---|
| `permaticker` | Int64 | Sharadar permanent ticker ID |
| `signal_date` | Date | End-of-month rebalance date |
| `in_estu` | Bool | ESTU membership on `signal_date` |
| `mcap` | Float64 | Market cap on `signal_date` (cap-weighting) |
| `log_mcap` | Float64 | Natural log of mcap (Size raw descriptor) |
| `size_z` | Float64 | Cross-sectionally standardised Size score (CW mean 0, EW std 1) |
| `size_z_cubed` | Float64 | size_z³ after 1%/99% winsorisation, before orthogonalisation |
| `nls_raw` | Float64 | Orthogonalised size_z³ (residual of WLS on size_z) |
| `nls_score` | Float64 | Final standardised exposure (the deliverable) |

---

## Shared Infrastructure

- Signal calendar: **330 dates** (1999-01-29 → 2026-06-10)
- ESTU size: mean **2,494**, min **1,791**, max **3,045**

---

## Known Deviations from PDF Spec

Mirrors the complete NOT-IN-PDF master list from `nls_spec.ipynb` (§9).

| # | Decision | Chosen solution | Revisit when |
|---|---|---|---|
| 1 | Winsorisation before cubing | 1%/99% cross-sectional EW quantile bounds per date | Standard USE4 convention; clip rate will be exactly 2% by construction |
| 2 | Orthogonalisation method | No-intercept WLS (√mcap weights, ESTU only) | If intercept is needed to break residual correlation |
| 3 | Size score source | Re-derived from log_mcap (not imported from size artifact) | Consistent; size_z here ≡ size_score from size_audit |
| 4 | Outlier trim | Standard 3σ trim applied to nls_raw | Clip rate will be ~2% (from winsorisation) not from fat tails |
| 5 | Non-ESTU standardisation | nlb_score applied universally using ESTU-derived parameters | Standard USE4 practice |
| 6 | LNCAP standardisation convention | CW mean 0, EW std 1 (matches Size factor) | Never — must match upstream Size convention |
| 7 | mcap source | ESTU artifact mcap | Never — signal_date mcap is always correct |
| 8 | Calendar | 330 dates (same as Size) | No deviation from Size calendar |

---

## Stage-by-Stage Audit

### Stage 1 — Log-mcap from ESTU

**Observed:**
- ESTU rows: **1,773,820**; **330** dates; **16,829** tickers

**Status:** ✅ **MATCHES SPEC**

### Stage 4 — NLS estimator

**PDF SPEC:** Winsorize size_z at 1%/99%, cube, orthogonalize to Size.

**Implementation:** Per date: 1%/99% EW winsorization of size_z; `size_z_cubed = winsor(size_z)^3`; no-intercept WLS on ESTU; `nls_raw = size_z_cubed − γ_t · size_z`.

**Observed:**
- Clip rate (by construction of 1%/99%): exactly **2.00%** of ESTU per date
- Pre-winsor orthogonality (max |Cov_CW(size_z³, size_z)|): **8.00×10⁻¹³** (target <10⁻⁶)
- Panel: **1,730,893** rows, **330** dates, **16,475** tickers; duplicates **0**
- Spot 2026-05-29: **4,926** tickers, median nls_raw **0.0099**

**Status:** ✅ **MATCHES SPEC**

### Stage 5 — Outlier trim (3σ)

**Implementation:** CW mean ± 3·EW std (ESTU bounds, applied to all).

**Observed:**
- Clipped (ESTU): ~2.00% (structural from winsorisation)
- Post-trim at 2026-05-29 (ESTU, n=2,761): mean 0.0119, std 2.0327, skew 0.002, kurt −0.391

**Status:** ✅ **MATCHES SPEC**

### Stage 6 — Standardisation

**Observed:**
- CW mean: **0.000000**; EW std: **1.000000**
- Fraction in [±3]: all **100.0%** / ESTU **100.0%** (signature of winsorise-then-standardise)

**Status:** ✅ **MATCHES SPEC**

### Stage 7 — Save

**Observed:**
- Rows **1,730,893**; dates **330**; tickers **16,475**; **52.2 MB**
- Schema check ✅ **PASSED**; read-back ✅ **PASSED**

**Status:** ✅ **MATCHES SPEC**

---

## Validation Checks

All checks passed on the run dated 2026-06-11.

| Check | Description | Observed | Target / Status |
|---|---|---|---|
| 1a | Max \|μ_CW(nls_score)\| | 2.64×10⁻¹⁶ | <10⁻⁶ ✅ PASS |
| 1b | Mean EW std | 1.0000 | 1.0±0.02 ✅ PASS |
| 2a | Pre-winsor orthogonality max \|Cov_CW\| | 8.00×10⁻¹³ | <10⁻⁶ ✅ PASS |
| 2b | Post-standardize max \|Cov_CW(nls, size)\| | <10⁻¹⁵ | <10⁻⁶ ✅ PASS |
| 3 | Fraction in [±3] | all 100.0% / ESTU 100.0% | 100% ✅ PASS |
| 4 | MoM Spearman ρ mean | 0.9882 | >0.92 ✅ PASS |
| 5 | ESTU clip rate | ~2.00% | 1.5–2.5% ✅ PASS |
| 6 | Disk vs memory max diff | 0.00×10⁰ | <10⁻¹⁰ ✅ PASS |

- **Check 3 note:** 100% in [±3] is expected — winsorisation at 1%/99% caps the cubic input, making extreme scores impossible. This is a diagnostic signature of the winsorise-then-standardise pipeline.

---

## Spot Check at 2026-05-29

- Coverage: **4,926** tickers; median nls_raw **0.0099**
- Post-trim ESTU (n=2,761): mean **0.0119**, std **2.0327**, skew **0.002**, excess kurtosis **−0.391**

> *No per-ticker factor values were printed by the build notebook; the spot check reports aggregate statistics only.*
