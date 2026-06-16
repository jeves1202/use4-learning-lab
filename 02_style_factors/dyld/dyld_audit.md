# Dividend Yield (DYLD) — Factor Construction Audit

**USE4-Faithful Implementation — `dyld_build.ipynb`**
Audit Date: 2026-06-11 | USE4 Learning Lab — Reference Audit

---

## Purpose and Scope

Audits the Dividend Yield factor: trailing 12-month dividends per share over the signal-date price (USE4 Empirical Notes, Appendix A). TTM DPS is the split-adjusted sum of the 4 most recent ARQ quarterly DPS filings with `datekey` ≤ `signal_date`.

**Scope.** This audit covers every pipeline stage of `dyld_build.ipynb` from the lab's reference run of 2026-06-11: all printed diagnostics, the validation battery, and the output parquet schema. All numerical values are verbatim from the executed notebook. The notebook generates no figures.

> *Reference numbers: every value in this document comes verbatim from the lab's reference run of 2026-06-11 (Sharadar data through 2026-06-10). Your rebuild will differ in row counts and tail statistics with data vintage; distribution shapes, stability coefficients, and check outcomes should match. Treat persistent sign or order-of-magnitude differences as bugs in your build until proven otherwise.*

---

## Output Schema

Deliverable: `data/out/dyld_use4.parquet`, zstd, statistics=True.

| Column | Dtype | Description |
|---|---|---|
| `permaticker` | Int64 | Sharadar permanent ticker ID |
| `signal_date` | Date | End-of-month rebalance date |
| `in_estu` | Bool | ESTU membership on `signal_date` |
| `mcap` | Float64 | Market cap on `signal_date` (cap-weighting) |
| `dyld` | Float64 | Raw descriptor: TTM DPS / PIT price (per share) |
| `dyld_score` | Float64 | Final standardised exposure (the deliverable) |
| `n_obs` | UInt32 | Number of non-null DPS quarters in the TTM window (1–4) |

---

## Shared Infrastructure

- Signal calendar: **330 dates** (1999-01-29 → 2026-06-10)
- ESTU size: mean **2,494**, min **1,791**, max **3,045**

---

## Known Deviations from PDF Spec

Mirrors the complete NOT-IN-PDF master list from `dyld_spec.ipynb` (§13).

| # | Decision | Chosen solution | Revisit when |
|---|---|---|---|
| 1 | TTM construction | 4-quarter rolling sum of ARQ DPS (datekey PIT) | When ARY annual data is available in Sharadar |
| 2 | Zero-dividend handling | Retain `dyld = 0.0` (valid exposure for non-payers) | If factor is used in a dividend-focused sub-model |
| 3 | Missing DPS (null) | Null `dyld` if n_obs=0; excluded from standardisation | If data quality improves and null reliably means "no dividend" |
| 4 | Negative DPS clamp | Clamp quarterly DPS to 0 before summing | Low impact; Sharadar rarely has negative DPS |
| 5 | PIT filter | `datekey ≤ signal_date` (filing date) | Not revisit — datekey is always correct for PIT |
| 6 | Minimum quarters | `MIN_QUARTERS = 1` | If coverage is too broad and includes very thin-history names |
| 7 | Outlier trim | Standard 3σ; no hard yield cap | If extreme REIT/MLP outliers persist after 3σ trim |

---

## Stage-by-Stage Audit

### Stage 1 — Splits + SF1 ARQ DPS → TTM

**PDF SPEC:** Trailing 12-month dividends per share.

**Observed:**
- Split events: **13,105** (7,053 tickers); reverse splits 6,204
- ARQ rows: **659,555**; corrupt single-quarter DPS nulled: **1,292**
- TTM rows: **608,164**; **15,964** tickers; negative TTM DPS: **0**

**Status:** ✅ **MATCHES SPEC**

### Stage 2 — Load shared ESTU

**Observed:**
- Signal calendar: **330 dates** (1999-01-29 → 2026-06-10)
- ESTU universe rows: **1,773,820**
- ESTU size: mean **2,494**, min **1,791**, max **3,045**

**Status:** ✅ **MATCHES SPEC**

### Stage 3/4 — PIT price join + DYLD estimator

**PDF SPEC:** Divide by current price per share.

**Implementation:** PIT price (closeunadj) joined per `signal_date`; `dyld` = TTM DPS / price; TTM economic ceiling (`dyld` > 0.5 → null).

**Observed:**
- Price rows: **37,119,713**; price-hit coverage **100.0%**
- DPS-hit rows: **1,642,915** (92.6%); panel **1,642,915** rows, **15,704** tickers; duplicates **0**
- Nulled by economic ceiling (all dates): **8,700**; negative dyld: **0**
- Spot 2026-05-29: **4,712** tickers; zero-payers **2,967** (63.0%)

**Status:** ✅ **MATCHES SPEC**

### Stage 5 — Outlier trim (3σ)

**Implementation:** CW mean ± 3·EW std (ESTU bounds, applied to all); lower bound rarely binds (DYLD ≥ 0).

**Observed:**
- Clipped (ESTU): **14,103** / 815,407 = **1.73%** (target 0.5–3%)
- Bounds: lo mean −0.0874; hi mean 0.1212 (max 0.2020)
- Post-trim at 2026-05-29 (ESTU, n=2,759): mean 0.0151, std 0.0233, skew 1.962, kurt 3.603

**Status:** ✅ **MATCHES SPEC**

### Stage 6 — Standardisation

**Observed:**
- CW mean: **0.000000**; EW std: **1.000000**
- Fraction in [±3]: all **97.2%** / ESTU **97.0%**

**Status:** ✅ **MATCHES SPEC**

### Stage 7 — Save

**Observed:**
- Rows **1,642,915**; dates **330**; tickers **15,704**; **14.1 MB**
- Schema check ✅ **PASSED**; read-back ✅ **PASSED**

**Status:** ✅ **MATCHES SPEC**

---

## Validation Checks

All checks passed on the run dated 2026-06-11.

| Check | Description | Observed | Target / Status |
|---|---|---|---|
| 1a | Max \|μ_CW(dyld_score)\| | 1.50×10⁻¹⁶ | <10⁻⁶ ✅ PASS |
| 1b | Mean EW std | 1.0000 | 1.0±0.02 ✅ PASS |
| 2a | min(dyld) | 0.000000 | ≥0.0 ✅ PASS |
| 2b | ESTU median dyld | 0.00285 | [0.000, 0.030] ✅ PASS |
| 2c | Max dyld post-trim | 0.2020 | ≤0.5 ✅ PASS |
| 3 | Min coverage post-2005 (completed months) | 4,277 | ≥4,000 ✅ PASS |
| 4 | MoM Spearman ρ mean | 0.9927 | >0.90 ✅ PASS |
| 5 | Q1→Q5 mean dyld monotone | spread 0.05579 | monotone ✅ PASS |
| 6 | Disk vs memory max diff | 0.00×10⁰ | <10⁻¹⁰ ✅ PASS |

- **Check 5 quintile means:** Q1 = 0.00000, Q2 = 0.00000, Q3 = 0.00352, Q4 = 0.01693, Q5 = 0.05579 (Q1/Q2 are the zero-payer mass).

---

## Spot Check at 2026-05-29

- Coverage: **4,712** tickers; median dyld **0.0000** (63.0% zero-payers)
- Post-trim ESTU (n=2,759): mean **0.0151**, std **0.0233**, skew **1.962**, excess kurtosis **3.603**

> *No per-ticker factor values were printed by the build notebook; the spot check reports aggregate statistics only.*
