# Liquidity (LIQ) — Factor Construction Audit

**USE4-Faithful Implementation — `liq_build.ipynb`**
Audit Date: 2026-06-11 | USE4 Learning Lab — Reference Audit

---

## Purpose and Scope

Audits the Liquidity factor: LIQ = 0.35·STOM + 0.35·STOQ + 0.30·STOA (USE4 Empirical Notes, Appendix A) — log share turnover over one month, the average of 3 months, and the average of 12 months.

**Scope.** This audit covers every pipeline stage of `liq_build.ipynb` from the lab's reference run of 2026-06-11: all printed diagnostics, the validation battery, and the output parquet schema. All numerical values are verbatim from the executed notebook. The notebook generates no figures.

> *Reference numbers: every value in this document comes verbatim from the lab's reference run of 2026-06-11 (Sharadar data through 2026-06-10). Your rebuild will differ in row counts and tail statistics with data vintage; distribution shapes, stability coefficients, and check outcomes should match. Treat persistent sign or order-of-magnitude differences as bugs in your build until proven otherwise.*

---

## Output Schema

Deliverable: `data/out/liq_use4.parquet`, zstd, statistics=True.

| Column | Dtype | Description |
|---|---|---|
| `permaticker` | Int64 | Sharadar permanent ticker ID |
| `signal_date` | Date | End-of-month rebalance date |
| `in_estu` | Bool | ESTU membership on `signal_date` |
| `mcap` | Float64 | Market cap on `signal_date` (cap-weighting) |
| `liq` | Float64 | Raw descriptor: 0.35·STOM + 0.35·STOQ + 0.30·STOA |
| `liq_score` | Float64 | Final standardised exposure (the deliverable) |
| `n_obs` | UInt32 | 1 if STOM and the minimum month requirements were met |

---

## Shared Infrastructure

- Signal calendar: **330 dates** (1999-01-29 → 2026-06-10)
- ESTU size: mean **2,494**, min **1,791**, max **3,045**

---

## Known Deviations from PDF Spec

Mirrors the complete NOT-IN-PDF master list from `liq_spec.ipynb` (§12).

| # | Decision | Chosen solution | Revisit when |
|---|---|---|---|
| 1 | Zero-volume days (V_t = 0) | Include in sum as V_t/S_t = 0 | Stage 4 — zero trading is genuine illiquidity; including it is economically correct |
| 2 | Fully-suspended month (all-zero volume) | STOM_τ = null | Stage 4 — null propagates cleanly; arbitrary floor creates scale distortion |
| 3 | Minimum valid windows for STOQ | 2 of 3 | Stage 4 — 2/3 allows recent IPOs to contribute a STOQ signal |
| 4 | Minimum valid windows for STOA | 2 of 12 | Stage 4 — 2/12 is permissive; tradeoff between coverage and signal quality |
| 5 | Composite with partial descriptors | Renormalise available weights to 1.0 | Stage 4 — renormalisation preserves coverage for newer stocks |
| 6 | Daily shares outstanding source | Sharadar SEP `sharesbas` | Stage 1 — SEP daily is more granular; captures intra-quarter share changes |
| 7 | Units for V_t/S_t computation | `volume / (sharesbas * 1e6)` | Stage 1 — **must verify empirically** before running; see Pitfall 3 |
| 8 | Trading day calendar | Dates present in Sharadar SEP | Stage 1 — SEP-native calendar is automatically PIT-consistent and avoids extra dependency |
| 9 | Outlier treatment | 3σ cap-weighted trim (USE4 standard) | Stage 5 — standard USE4 default applied to all descriptors |
| 10 | Minimum obs for composite inclusion | MIN_OBS = 1 (any valid STOM window) | Stage 4 — single STOM allows maximum coverage; raise if signal quality concerns emerge |

---

## Stage-by-Stage Audit

### Stage 1 — SEP turnover

**PDF SPEC:** Share turnover from daily volume and shares outstanding.

**Observed:**
- SEP rows: **36,343,678**; **16,624** tickers; **6,921** trading days
- Turnover fraction (2020+, positive days): median **0.00461** (target 0.001–0.010) ✅ unit check OK
- Impossible turnover (>1): **20,710** rows (0.057%), 2,221 tickers

**Status:** ✅ **MATCHES SPEC**

### Stage 2 — Load shared ESTU

**Observed:**
- Signal calendar: **330 dates** (1999-01-29 → 2026-06-10)
- ESTU universe rows: **1,773,820**
- ESTU size: mean **2,494**, min **1,791**, max **3,045**

**Status:** ✅ **MATCHES SPEC**

### Stage 4 — LIQ estimator

**PDF SPEC:** Weighted composite of log turnover at 1/3/12-month horizons.

**Implementation:** STOM = ln(Σ_21d); STOQ = ln(mean_3m); STOA = ln(mean_12m); weights 0.35/0.35/0.30.

**Observed:**
- Raw panel: **1,746,700** rows, **330** dates, **16,574** tickers; duplicates **0**
- Spot 2026-05-29: **4,925** tickers, median liq **−1.633**; range [−9.935, 8.593]

**Status:** ✅ **MATCHES SPEC**

### Stage 5 — Outlier trim (3σ)

**Implementation:** CW mean ± 3·EW std (ESTU bounds, applied to all).

**Observed:**
- Clipped (ESTU): **7,785** / 822,918 = **0.95%** (target 0.5–2%)
- Bounds: lo mean −4.494; hi mean 0.774 (max 1.564)
- Post-trim at 2026-05-29 (ESTU, n=2,761): mean −1.476, std 0.729, skew −0.057, kurt 0.608

**Status:** ✅ **MATCHES SPEC**

### Stage 6 — Standardisation

**Observed:**
- CW mean: **0.000000**; EW std: **1.000000**
- Fraction in [±3]: all **84.3%** / ESTU **98.9%** (923,782 non-ESTU illiquid names pull the all-universe tail)

**Status:** ✅ **MATCHES SPEC**

### Stage 7 — Save

**Observed:**
- Rows **1,746,700**; dates **330**; tickers **16,574**; **30.2 MB**
- Schema check ✅ **PASSED**; read-back ✅ **PASSED**

**Status:** ✅ **MATCHES SPEC**

---

## Validation Checks

All checks passed on the run dated 2026-06-11.

| Check | Description | Observed | Target / Status |
|---|---|---|---|
| 1a | Max \|μ_CW(liq_score)\| | 6.57×10⁻¹⁶ | <10⁻⁶ ✅ PASS |
| 1b | Mean EW std | 1.0000 | 1.0±0.02 ✅ PASS |
| 2 | liq in [−6, 2] for ESTU | 100.0% [−5.37, 1.56] | ≥95% ✅ PASS |
| 3 | Min coverage post-2005 (completed months) | 4,636 | ≥4,000 ✅ PASS |
| 4 | MoM Spearman ρ mean | 0.9706 | >0.95 ✅ PASS |
| 5 | Q1→Q5 mean liq monotone | spread 2.3483 | monotone ✅ PASS |
| 6 | Disk vs memory max diff | 0.00×10⁰ | <10⁻¹⁰ ✅ PASS |

- **Check 5 quintile means:** Q1 = −2.9928, Q2 = −2.2370, Q3 = −1.8509, Q4 = −1.4397, Q5 = −0.6445 (log turnover is negative by construction).

---

## Spot Check at 2026-05-29

- Coverage: **4,925** tickers; median liq **−1.633**
- Post-trim ESTU (n=2,761): mean **−1.476**, std **0.729**, skew **−0.057**, excess kurtosis **0.608**

> *No per-ticker factor values were printed by the build notebook; the spot check reports aggregate statistics only.*
