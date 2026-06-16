# Book-to-Price (BP) — Factor Construction Audit

**USE4-Faithful Implementation — `bp_build.ipynb`**
Audit Date: 2026-06-11 | USE4 Learning Lab — Reference Audit

---

## Purpose and Scope

Audits the Book-to-Price factor: BP = 1.0 · BTOP, BTOP_i,t = `equity`_i,t / `mcap`_i,t (USE4 Empirical Notes, Appendix A, p. 52) — last reported book value of common equity (Sharadar SF1, ARQ) over signal-date market capitalisation.

**Scope.** This audit covers every pipeline stage of `bp_build.ipynb` from the lab's reference run of 2026-06-11: all printed diagnostics, the validation battery, and the output parquet schema. All numerical values are verbatim from the executed notebook. The notebook generates no figures.

> *Reference numbers: every value in this document comes verbatim from the lab's reference run of 2026-06-11 (Sharadar data through 2026-06-10). Your rebuild will differ in row counts and tail statistics with data vintage; distribution shapes, stability coefficients, and check outcomes should match. Treat persistent sign or order-of-magnitude differences as bugs in your build until proven otherwise.*

---

## Output Schema

Deliverable: `data/out/bp_use4.parquet`, zstd, statistics=True.

| Column | Dtype | Description |
|---|---|---|
| `permaticker` | Int64 | Sharadar permanent ticker ID |
| `signal_date` | Date | End-of-month rebalance date |
| `in_estu` | Bool | ESTU membership on `signal_date` |
| `mcap` | Float64 | Market cap on `signal_date` (cap-weighting) |
| `btop` | Float64 | Raw descriptor: equity/mcap |
| `btop_score` | Float64 | Final standardised exposure (the deliverable) |
| `n_obs` | UInt32 | 1 if a valid ARQ filing within the lookback window |

---

## Shared Infrastructure

- Signal calendar: **330 dates** (1999-01-29 → 2026-06-10)
- ESTU size: mean **2,494**, min **1,791**, max **3,045**

---

## Known Deviations from PDF Spec

Mirrors the complete NOT-IN-PDF master list from `bp_spec.ipynb` (§13).

| # | Decision | Chosen solution | Revisit when |
|---|---|---|---|
| 1 | Common equity definition ⚠️ **Material** | `equity` (no PREFEQUITY in Sharadar) | If a preferred equity balance-sheet field is added to data pipeline |
| 2 | SF1 dimension | ARQ (as-reported quarterly) — book value is a balance-sheet stock, not a flow | Never — ARQ is always correct for balance-sheet items |
| 3 | PIT alignment | `datekey` (filing date) as PIT constraint | If Sharadar datekey quality is suspected |
| 4 | ME denominator source | `estu_monthly.mcap` on signal_date | Never — estu_monthly mcap is the standard source for all factors |
| 5 | Staleness cutoff | 548 calendar days (~18 months) — same as EYLD | If stale book values are suspected to add noise |
| 6 | Negative BE treatment | Retain as valid negative BTOP | Only if factor is used in an application where negative book equity is meaningless |
| 7 | Zero/missing ME | Mark BTOP as null; exclude from computation | Never — zero or missing mcap is a data error, not a valid observation |
| 8 | Outlier trim multiplier | 3σ (PDF default) | Stage 5 |
| 9 | MIN_OBS threshold | 1 (binary: has valid ARQ or not) | If n_obs proves uninformative |
| 10 | Calendar start | 1999-01-01 | If early data quality is poor |

---

## Stage-by-Stage Audit

### Stage 1 — SF1 ARQ fundamentals

**PDF SPEC:** Last reported book value of common equity, point-in-time.

**Observed:**
- ARQ rows: **608,786**; **15,964** tickers (1999-01-04 → 2026-06-10)
- Null equity: **224** (0.0%; target < 10%); negative equity: **49,043** rows

**Status:** ✅ **MATCHES SPEC**

### Stage 2 — Load shared ESTU

**Observed:**
- Signal calendar: **330 dates** (1999-01-29 → 2026-06-10)
- ESTU universe rows: **1,773,820**
- ESTU size: mean **2,494**, min **1,791**, max **3,045**

**Status:** ✅ **MATCHES SPEC**

### Stage 4 — BP estimator (PIT join)

**PDF SPEC:** BTOP = last reported BE / current market cap.

**Implementation:** `join_asof` backward on `datekey` ≤ `signal_date`; staleness ≤ 548 days; `mcap` > 0 guard; `btop` = `equity`/`mcap`.

**Observed:**
- Rows with SF1 hit: **1,643,446** of 1,773,820
- Panel: **1,635,030** rows, **330** dates, **15,703** tickers; duplicates **0**
- Spot 2026-05-29: **4,705** tickers, median btop **0.3875**

**Status:** ✅ **MATCHES SPEC**

### Stage 5 — Outlier trim (3σ)

**Implementation:** CW mean ± 3·EW std per date (ESTU), clip applied to all stocks.

**Observed:**
- Clipped (ESTU): **11,644** / 817,361 = **1.42%** (target 0.5–2%)
- Bounds: lo mean −1.211 (min −10.064); hi mean 1.905 (max 10.828)
- Post-trim at 2026-05-29 (ESTU, n=2,759): mean 0.446, std 0.408, skew 0.635, kurt 0.987

**Status:** ✅ **MATCHES SPEC**

### Stage 6 — Standardisation

**Observed:**
- CW mean: **0.000000**; EW std: **1.000000**
- Fraction in [±3]: all **89.8%** / ESTU **97.5%**

**Status:** ✅ **MATCHES SPEC**

### Stage 7 — Save

**Observed:**
- Rows **1,635,030**; dates **330**; tickers **15,703**; **28.2 MB**
- Schema check ✅ **PASSED**; read-back ✅ **PASSED**

**Status:** ✅ **MATCHES SPEC**

---

## Validation Checks

All checks passed on the run dated 2026-06-11.

| Check | Description | Observed | Target / Status |
|---|---|---|---|
| 1a | Max \|μ_CW(btop_score)\| | 2.21×10⁻¹⁶ | <10⁻⁶ ✅ PASS |
| 1b | Mean EW std | 1.0000 | 1.0±0.02 ✅ PASS |
| 2a | Median btop (ESTU, post-2005) | 0.4077 | [0.20, 0.80] ✅ PASS |
| 2b | Negative btop present (post-2000) | confirmed | present ✅ PASS |
| 2c | Max btop post-trim | 10.8275 | <15.0 ✅ PASS |
| 3 | Min coverage post-2005 (completed months) | 4,330 | ≥4,000 ✅ PASS |
| 4 | MoM Spearman ρ mean | 0.9773 | >0.97 ✅ PASS |
| 5 | Q1→Q5 mean btop monotone | spread 1.01947 | monotone ✅ PASS |
| 6 | Disk vs memory max diff | 0.00×10⁰ | <10⁻¹⁰ ✅ PASS |

- **Check 2c:** the <15 bound guards against unit mismatch; the observed max 10.8275 is the legitimate 3σ clip boundary on the widest-dispersion date.
- **Check 5 quintile means:** Q1 = 0.04308, Q2 = 0.24736, Q3 = 0.40271, Q4 = 0.60874, Q5 = 1.06255.

---

## Spot Check at 2026-05-29

- Coverage: **4,705** tickers; median btop **0.3875**
- Post-trim ESTU (n=2,759): mean **0.446**, std **0.408**, skew **0.635**, excess kurtosis **0.987**

> *No per-ticker factor values were printed by the build notebook; the spot check reports aggregate statistics only.*
