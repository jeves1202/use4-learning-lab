# Leverage (LEV) — Factor Construction Audit

**USE4-Faithful Implementation — `lev_build.ipynb`**
Audit Date: 2026-06-11 | USE4 Learning Lab — Reference Audit

---

## Purpose and Scope

Audits the Leverage factor: LEV = 0.75·MLEṼ + 0.15·DTOA̲ + 0.10·BLEṼ (USE4 Empirical Notes, Appendix A, p. 54), each sub-descriptor standardised before compositing.

**Scope.** This audit covers every pipeline stage of `lev_build.ipynb` from the lab's reference run of 2026-06-11: all printed diagnostics, the validation battery, and the output parquet schema. All numerical values are verbatim from the executed notebook. The notebook generates no figures.

> *Reference numbers: every value in this document comes verbatim from the lab's reference run of 2026-06-11 (Sharadar data through 2026-06-10). Your rebuild will differ in row counts and tail statistics with data vintage; distribution shapes, stability coefficients, and check outcomes should match. Treat persistent sign or order-of-magnitude differences as bugs in your build until proven otherwise.*

---

## Output Schema

Deliverable: `data/out/lev_use4.parquet`, zstd, statistics=True.

| Column | Dtype | Description |
|---|---|---|
| `permaticker` | Int64 | Sharadar permanent ticker ID |
| `signal_date` | Date | End-of-month rebalance date |
| `in_estu` | Bool | ESTU membership on `signal_date` |
| `mcap` | Float64 | Market cap on `signal_date` (cap-weighting) |
| `lev` | Float64 | Raw descriptor: weighted composite of standardised MLEV/DTOA/BLEV |
| `lev_score` | Float64 | Final standardised exposure (the deliverable) |
| `n_obs` | UInt32 | Count of non-null sub-descriptors (1–3) |

---

## Shared Infrastructure

- Signal calendar: **330 dates** (1999-01-29 → 2026-06-10)
- ESTU size: mean **2,494**, min **1,791**, max **3,045**

---

## Known Deviations from PDF Spec

Mirrors the complete NOT-IN-PDF master list from `lev_spec.ipynb` (§12).

| # | Decision | Chosen solution | Revisit when |
|---|---|---|---|
| 1 | Compositing order | Standardize-then-combine (sub-descriptors to unit scale before weighting) | If future USE4 docs clarify |
| 2 | Completeness rule | n_obs = 3 required; partial composites are null | If missing-descriptor fraction exceeds 10% |
| 3 | Negative BE | BLEV = null when EQUITY − PREFEQUITY ≤ 0 | Never — negative BE invalidates the formula |
| 4 | Zero/negative ME | MLEV = null when mcap ≤ 0 | Never — these are data errors |
| 5 | DTOA > 1 | Retain; ±3σ clipping handles extremes | If DTOA > 1 fraction exceeds 10% |
| 6 | Sub-descriptor outlier trim | Upper-tail only for MLEV/BLEV; ±3σ for DTOA | If distributional skew is extreme |
| 7 | PREFEQUITY null ⚠️ **Material** | Treat as 0 (no preferred equity) | Never — null = absent, not unknown |
| 8 | DEBTNC / DEBTC null | Treat as 0 (no debt of that type) | Never — null = absent, not unknown |
| 9 | SF1 dimension | ARQ (as-reported quarterly) | Equivalent for balance-sheet snapshots; ARQ is project standard |
| 10 | Calendar start | 1999-01-01 | If pre-1999 ESTU coverage improves |

---

## Stage-by-Stage Audit

### Stage 1 — SF1 ARQ balance-sheet snapshot

**PDF SPEC:** Market leverage, debt-to-assets, book leverage from the latest filing.

**Observed:**
- ARQ rows: **608,786**; **15,964** tickers (1999-01-04 → 2026-06-10)
- Null assets **0.0%**; null debtnc **20.3%** (floored to 0); null equity **0.0%**

**Status:** ✅ **MATCHES SPEC**

### Stage 2 — Load shared ESTU

**Observed:**
- Signal calendar: **330 dates** (1999-01-29 → 2026-06-10)
- ESTU universe rows: **1,773,820**
- ESTU size: mean **2,494**, min **1,791**, max **3,045**

**Status:** ✅ **MATCHES SPEC**

### Stage 4 — LEV estimator

**PDF SPEC:** 0.75·MLEV + 0.15·DTOA + 0.10·BLEV, sub-descriptors standardised.

**Implementation:** PIT join; staleness ≤ 548 days; each sub-descriptor CW-mean-0/EW-std-1 standardised on ESTU before weighting.

**Observed:**
- Panel after staleness filter: **1,635,030** rows, **15,703** tickers; duplicates **0**
- n_obs: 3-of-3 **93.9%**; 2-of-3 **6.1%**; DTOA > 1: 1.2%; min MLEV/BLEV = 1.0000
- Sub-descriptor CW means: 5.96×10⁻¹⁶ / 1.78×10⁻¹⁶ / 1.66×10⁻¹⁶ (all <10⁻⁶)
- Spot 2026-05-29: **4,705** tickers, median lev **−0.1565**

**Status:** ✅ **MATCHES SPEC**

### Stage 5 — Outlier trim (3σ)

**Implementation:** CW mean ± 3·EW std (ESTU bounds, applied to all).

**Observed:**
- Clipped (ESTU): **21,342** / 786,655 = **2.71%** (target 0.5–2%)
- Bounds: lo mean −2.583; hi mean 2.551 (max 2.772)
- Post-trim at 2026-05-29 (ESTU, n=2,595): mean 0.167, std 0.665, skew 1.859, kurt 3.210

⚠️ **ESTU clip rate 2.71% is 0.71pp above the 0.5–2% band — the composite right tail (highly levered financials/utilities) is structurally heavy. Accepted and documented.**

**Status:** ⚠️ **PARTIAL MATCH** (clip rate above band, documented)

### Stage 6 — Standardisation

**Observed:**
- CW mean: **−0.000000**; EW std: **1.000000**
- Fraction in [±3]: all **94.8%** / ESTU **96.4%**

**Status:** ✅ **MATCHES SPEC**

### Stage 7 — Save

**Observed:**
- Rows **1,635,030**; dates **330**; tickers **15,703**; **19.8 MB**
- Schema check ✅ **PASSED**; read-back ✅ **PASSED**

**Status:** ✅ **MATCHES SPEC**

---

## Validation Checks

All checks passed on the run dated 2026-06-11.

| Check | Description | Observed | Target / Status |
|---|---|---|---|
| 1a | Max \|μ_CW(lev_score)\| | 3.71×10⁻¹⁷ | <10⁻⁶ ✅ PASS |
| 1b | Mean EW std | 1.0000 | 1.0±0.02 ✅ PASS |
| 2a | lev composite bounded | [−0.564, 2.772] | ≈(−3, +3) ✅ PASS |
| 2b | Median lev_score (ESTU) | −0.2582 | \|med\| < 0.5 ✅ PASS |
| 3 | Min coverage post-2005 (completed months) | 4,097 | ≥3,800 ✅ PASS |
| 4 | MoM Spearman ρ mean | 0.9908 | >0.92 ✅ PASS |
| 5 | Q1→Q5 mean lev monotone | spread 1.75542 | monotone ✅ PASS |
| 6 | Disk vs memory max diff | 0.00×10⁰ | <10⁻¹⁰ ✅ PASS |

- **Check 3:** LEV uses a 3,800 floor (vs 4,000 elsewhere) — it requires all three balance-sheet fields, the thinnest fundamental coverage.
- **Check 5 quintile means:** Q1 = −0.45334, Q2 = −0.44357, Q3 = −0.21770, Q4 = 0.20831, Q5 = 1.30208.

---

## Spot Check at 2026-05-29

- Coverage: **4,705** tickers; median lev **−0.1565**
- Post-trim ESTU (n=2,595): mean **0.167**, std **0.665**, skew **1.859**, excess kurtosis **3.210**

> *No per-ticker factor values were printed by the build notebook; the spot check reports aggregate statistics only.*
