# Size (SIZE) — Factor Construction Audit

**USE4-Faithful Implementation — `size_build.ipynb`**
Audit Date: 2026-06-11 | USE4 Learning Lab — Reference Audit

---

## Purpose and Scope

Audits the Size factor: SIZE = 1.0·LNCAP = ln(mcap), the natural log of market capitalisation on the signal date (USE4 Empirical Notes, Appendix A). This is the simplest factor in the pipeline: one raw descriptor, no fundamental data.

**Scope.** This audit covers every pipeline stage of `size_build.ipynb` from the lab's reference run of 2026-06-11: all printed diagnostics, the validation battery, and the output parquet schema. All numerical values are verbatim from the executed notebook. The notebook generates no figures.

> *Reference numbers: every value in this document comes verbatim from the lab's reference run of 2026-06-11 (Sharadar data through 2026-06-10). Your rebuild will differ in row counts and tail statistics with data vintage; distribution shapes, stability coefficients, and check outcomes should match. Treat persistent sign or order-of-magnitude differences as bugs in your build until proven otherwise.*

---

## Output Schema

Deliverable: `data/out/size_use4.parquet`, zstd, statistics=True.

| Column | Dtype | Description |
|---|---|---|
| `permaticker` | Int64 | Sharadar permanent ticker ID |
| `signal_date` | Date | End-of-month rebalance date |
| `in_estu` | Bool | ESTU membership on `signal_date` |
| `mcap` | Float64 | Market cap on `signal_date` (cap-weighting) |
| `log_mcap` | Float64 | Raw descriptor: ln(mcap) |
| `size_score` | Float64 | Final standardised exposure (the deliverable) |
| `n_obs` | UInt32 | Always 1 if log_mcap is valid; 0 if mcap is null/zero |

---

## Shared Infrastructure

- Signal calendar: **330 dates** (1999-01-29 → 2026-06-10)
- ESTU size: mean **2,494**, min **1,791**, max **3,045**

---

## Known Deviations from PDF Spec

Mirrors the complete NOT-IN-PDF master list from `size_spec.ipynb` (§9).

| # | Decision | Chosen solution | Revisit when |
|---|---|---|---|
| 1 | Float vs total mcap ⚠️ **Material** | Total mcap (closeunadj × sharesbas × 1e6) | When float-adjusted share counts become available in Sharadar |
| 2 | Trim convention | 4σ EW-centered trim (not the family-standard 3σ CW-centered) | If USE4 source clarifies; 4σ EW is the USE4 Barra specification for LNCAP |
| 3 | mcap source | ESTU artifact mcap (not re-derived from SEP) | Never — ESTU artifact mcap is the project's authoritative mcap |
| 4 | Zero/negative mcap | Excluded (n_obs=0); effectively null | Never — these are data errors |
| 5 | Calendar start | 1999-01-01 | If pre-1999 ESTU coverage improves |
| 6 | mcap units | USD millions (Sharadar convention × 1e6 already applied in ESTU artifact) | Never — units are fixed by the ESTU artifact |
| 7 | 4σ vs 3σ | 4σ with EW-centered: USE4/Barra prescribes wider trim for LNCAP specifically | Never revisit — this deviation is spec-faithful |
| 8 | All-universe fraction in [±3] | Only 44.4% (structural: mega-cap CW mean pulls small-cap scores negative) | Structural; not a data-quality issue |

---

## Stage-by-Stage Audit

### Stage 1 — Log-mcap from ESTU

**PDF SPEC:** LNCAP = ln(market capitalisation).

**Observed:**
- ESTU rows: **1,773,820**; **330** dates; **16,829** tickers
- log_mcap range: [16.1, 28.3]; null mcap: **0**

**Status:** ✅ **MATCHES SPEC**

### Stage 5 — Outlier trim (4σ, EW-centered)

**Implementation:** EW mean ± 4·EW std per date (not CW-centered 3σ — this is the USE4 Barra specification for LNCAP).

**Observed:**
- Clipped (ESTU): **408** / 822,918 = **0.05%** (target <1%)
- Bounds: lo mean 18.02; hi mean 24.77
- Post-trim at 2026-05-29 (ESTU, n=2,761): mean 21.406, std 1.659, skew 0.030, kurt −0.477

**Status:** ✅ **MATCHES SPEC**

### Stage 6 — Standardisation

**Observed:**
- CW mean: **0.000000**; EW std: **1.000000**
- Fraction in [±3]: all **44.4%** / ESTU **97.1%**

> *44.4% all-universe in [±3] is structural: the ESTU CW mean is dominated by mega-caps (~$80B+), making most micro-caps score well below −3. This is not a data error.*

**Status:** ✅ **MATCHES SPEC**

### Stage 7 — Save

**Observed:**
- Rows **1,773,820**; dates **330**; tickers **16,829**; **25.8 MB**
- Schema check ✅ **PASSED**; read-back ✅ **PASSED**

**Status:** ✅ **MATCHES SPEC**

---

## Validation Checks

All checks passed on the run dated 2026-06-11.

| Check | Description | Observed | Target / Status |
|---|---|---|---|
| 1a | Max \|μ_CW(size_score)\| | 6.12×10⁻¹⁷ | <10⁻⁶ ✅ PASS |
| 1b | Mean EW std | 1.0000 | 1.0±0.02 ✅ PASS |
| 2a | ESTU median log_mcap (mean of medians) | ~21.4 | [19.0, 25.0] ✅ PASS |
| 2b | All-universe fraction in [±3] | 44.4% | >30% ✅ PASS |
| 3 | Min coverage post-2005 (completed months) | 4,614 | ≥4,000 ✅ PASS |
| 4 | MoM Spearman ρ mean | 0.9967 | >0.99 ✅ PASS |
| 5 | Q1→Q5 mean log_mcap monotone | spread 3.8866 | monotone ✅ PASS |
| 6 | Disk vs memory max diff | 0.00×10⁰ | <10⁻¹⁰ ✅ PASS |

- **Check 5 quintile means (log_mcap):** Q1 = 19.8945, Q2 = 20.6055, Q3 = 21.3155, Q4 = 22.1411, Q5 = 23.7810.
- **Check 4:** MoM ρ = 0.9967 is the highest of all 12 style factors — log-mcap is highly persistent.

---

## Spot Check at 2026-05-29

- Coverage: **4,926** tickers; ESTU **2,761** stocks
- Post-trim ESTU: mean **21.406**, std **1.659**, skew **0.030**, excess kurtosis **−0.477**

> *No per-ticker factor values were printed by the build notebook; the spot check reports aggregate statistics only.*
