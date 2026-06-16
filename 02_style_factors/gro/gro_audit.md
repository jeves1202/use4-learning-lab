# Growth (GRO) — Factor Construction Audit

**USE4-Faithful Implementation — `gro_build.ipynb`**
Audit Date: 2026-06-11 | USE4 Learning Lab — Reference Audit

---

## Purpose and Scope

Audits the Growth factor. USE4 weights (Appendix A, p. 52): 0.70·EGRLF + 0.20·EGRO + 0.10·SGRO; with no analyst-forecast data in Sharadar, EGRLF is proxied by EGRO, giving effective **GRO = 0.90·EGRO + 0.10·SGRO** — 5-year normalised OLS slopes of EPS and SPS.

**Scope.** This audit covers every pipeline stage of `gro_build.ipynb` from the lab's reference run of 2026-06-11: all printed diagnostics, the validation battery, and the output parquet schema. All numerical values are verbatim from the executed notebook. The notebook generates no figures.

> *Reference numbers: every value in this document comes verbatim from the lab's reference run of 2026-06-11 (Sharadar data through 2026-06-10). Your rebuild will differ in row counts and tail statistics with data vintage; distribution shapes, stability coefficients, and check outcomes should match. Treat persistent sign or order-of-magnitude differences as bugs in your build until proven otherwise.*

---

## Output Schema

Deliverable: `data/out/gro_use4.parquet`, zstd, statistics=True.

| Column | Dtype | Description |
|---|---|---|
| `permaticker` | Int64 | Sharadar permanent ticker ID |
| `signal_date` | Date | End-of-month rebalance date |
| `in_estu` | Bool | ESTU membership on `signal_date` |
| `mcap` | Float64 | Market cap on `signal_date` (cap-weighting) |
| `gro` | Float64 | Raw descriptor: 0.90·EGRO + 0.10·SGRO |
| `gro_score` | Float64 | Final standardised exposure (the deliverable) |
| `n_obs` | UInt32 | Annual observations in the regression (3–5) |

---

## Shared Infrastructure

- Signal calendar: **330 dates** (1999-01-29 → 2026-06-10)
- ESTU size: mean **2,494**, min **1,791**, max **3,045**

---

## Known Deviations from PDF Spec

Mirrors the complete NOT-IN-PDF master list from `gro_spec.ipynb` (§12).

| # | Decision | Chosen solution | Revisit when |
|---|---|---|---|
| 1 | EGRLF data unavailability ⚠️ **Material** | Proxy with EGRO; effective weights 0.90·EGRO + 0.10·SGRO | When analyst LTG forecast data (I/B/E/S, FactSet) becomes available in Sharadar |
| 2 | Negative mean EPS/SPS | Mark EGRO/SGRO null when mean(EPS) ≤ 0 or mean(SPS) ≤ 0 | When researching loss-making growth stocks (e.g. pre-profitability tech) |
| 3 | Minimum observations | Accept ≥ 3 annual data points (MIN_OBS = 3) | If coverage is sufficient without the 3-year floor |
| 4 | PIT filter | `datekey ≤ signal_date`; annual series derived from ARQ (ARY absent) | If ARY causes excessive data sparsity in early periods |
| 5 | One descriptor null | Renormalize available weights to sum to 1.0 | If renormalized GRO is found to distort cross-sectional ranking |

---

## Stage-by-Stage Audit

### Stage 1 — SF1 annual EPS/SPS

**PDF SPEC:** 5 years of annual earnings and sales per share.

**Observed:**
- ARQ rows **659,555**; annual rows **158,646**; **15,715** tickers
- EPS null rate **0.0%**; SPS null rate **0.0%**

**Status:** ✅ **MATCHES SPEC**

### Stage 2 — Load shared ESTU

**Observed:**
- Signal calendar: **330 dates** (1999-01-29 → 2026-06-10)
- ESTU universe rows: **1,773,820**
- ESTU size: mean **2,494**, min **1,791**, max **3,045**

**Status:** ✅ **MATCHES SPEC**

### Stage 4 — GRO estimator

**PDF SPEC:** Normalised 5-year OLS slope of EPS (and SPS).

**Implementation:** PIT cross-join of annual filings to signal dates; per-(stock, date) OLS slope / mean; EGRO/SGRO pre-clipped to ±5; composite 0.90/0.10.

**Observed:**
- PIT cross-join: **27,039,371** rows; aggregated (n≥3): **2,898,336**
- Pre-clipped: EGRO **41,815** rows, SGRO **1,673** rows
- Panel: **2,898,336** rows, **315** dates, **13,452** tickers; duplicates **0**; GRO null rate 1.8%
- Spot 2026-05-29: **13,445** tickers, median gro **0.0096**

**Status:** ✅ **MATCHES SPEC**

### Stage 5 — Outlier trim (3σ)

**Implementation:** CW mean ± 3·EW std (ESTU bounds, applied to all).

**Observed:**
- Clipped (ESTU): **18,507** / 674,821 = **2.74%** (target 0.5–2%)
- Bounds: lo mean −2.377; hi mean 2.128
- Post-trim at 2026-05-29 (ESTU, n=2,551): mean −0.0394, std 0.5622, skew −0.242, kurt 7.780

⚠️ **ESTU clip rate 2.74% sits 0.74pp above the 0.5–2% band — growth slopes are fat-tailed even after the ±5 pre-clip. Accepted: tightening the pre-clip would distort genuine high-growth names.**

**Status:** ⚠️ **PARTIAL MATCH** (clip rate above band, documented)

### Stage 6 — Standardisation

**Observed:**
- CW mean: **−0.000000**; EW std: **1.000000**
- Fraction in [±3]: all **95.5%** / ESTU **96.2%**

**Status:** ✅ **MATCHES SPEC**

### Stage 7 — Save

**Observed:**
- Rows **2,898,336**; dates **315** (2000-04-28 → 2026-06-10); tickers **13,452**; **50.5 MB**
- Schema check ✅ **PASSED**; read-back ✅ **PASSED**

**Status:** ✅ **MATCHES SPEC**

---

## Validation Checks

All checks passed on the run dated 2026-06-11.

| Check | Description | Observed | Target / Status |
|---|---|---|---|
| 1a | Max \|μ_CW(gro_score)\| | 7.97×10⁻¹⁷ | <10⁻⁶ ✅ PASS |
| 1b | Mean EW std | 1.0000 | 1.0±0.02 ✅ PASS |
| 2 | ESTU median gro (mean of medians) | −0.08120 | (−0.30, 0.50) ✅ PASS |
| 3 | Min coverage post-2005 (completed months) | 7,261 | ≥4,000 ✅ PASS |
| 4 | MoM Spearman ρ mean | 0.9863 | >0.95 ✅ PASS |
| 5 | Q1→Q5 mean gro monotone | spread 1.25906 | monotone ✅ PASS |
| 6 | Disk vs memory max diff | 0.00×10⁰ | <10⁻¹⁰ ✅ PASS |

- **Check 5 quintile means:** Q1 = −0.72398, Q2 = −0.18617, Q3 = −0.08073, Q4 = 0.02322, Q5 = 0.53508.
- **Coverage** is far above floor (7,261) because GRO retains every stock with ≥3 annual filings, a much wider net than price-based factors.

---

## Spot Check at 2026-05-29

- Coverage: **13,445** tickers; median gro **0.0096**
- Post-trim ESTU (n=2,551): mean **−0.0394**, std **0.5622**, skew **−0.242**, excess kurtosis **7.780**

> *No per-ticker factor values were printed by the build notebook; the spot check reports aggregate statistics only.*
