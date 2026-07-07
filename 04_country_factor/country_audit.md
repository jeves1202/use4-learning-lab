# Country Factor — Factor Construction Audit

**USE4-Faithful Implementation — `country_build.ipynb`**
Audit Date: 2026-06-12 | USE4 Learning Lab — Reference Audit

---

## Purpose and Scope

Audits the Country factor: a single US-market intercept constrained to be orthogonal to the industry factors via an exact reparametrisation (USE4 Empirical Notes §3, §6). In a single-country model the "country" return is identified by requiring the cap-weighted sum of industry factor returns to equal the CW intercept — achieved through a zero-sum linear constraint on industry exposures.

**Scope.** This audit covers every pipeline stage of `country_build.ipynb` from the lab's reference run of 2026-06-12: all printed diagnostics, the validation battery, and the output parquet schema. All numerical values are verbatim from the executed notebook.

> *Reference numbers: every value in this document comes verbatim from the lab's reference run of 2026-06-12 (Sharadar data through 2026-06-10). Your rebuild will differ in row counts and tail statistics with data vintage; distribution shapes, stability coefficients, and check outcomes should match. Treat persistent sign or order-of-magnitude differences as bugs in your build until proven otherwise.*

---

## Output Schema

### Anchor Artifact

Deliverable: `data/out/country_use4.parquet`, zstd, statistics=True.

| Column | Dtype | Description |
|---|---|---|
| `permaticker` | Int64 | Sharadar permanent ticker ID |
| `signal_date` | Date | End-of-month rebalance date |
| `in_estu` | Bool | Always True — the anchor contains in-ESTU rows only |
| `mcap` | Float64 | Market cap on `signal_date` (cap-weighting) |
| `country` | Float64 | Country factor exposure (always 1.0 for all stocks) |
| `w_reg` | Float64 | √mcap regression weight, normalized to sum to 1 per date over the ESTU |

### CSR Returns Artifact

Validation artifact (not a model deliverable): `data/out/csr_validation_returns.parquet`

| Column | Dtype | Description |
|---|---|---|
| `signal_date` | Date | Exposure date t (start of the return month) |
| `ret_date` | Date | End of the return month (the next signal date) |
| `factor` | Utf8 | Factor name (country, the 55 industry names, or the 12 style names) |
| `f` | Float64 | Factor return for the month (null if the column was degenerate that month) |
| `r2` | Float64 | Weighted cross-sectional R² of the month's regression (repeated per row) |
| `n_stocks` | UInt32 | Number of stocks in the month's regression (repeated per row) |

> *Note: the production CSR (`05_csr/csr_spec.ipynb`) applies stricter sample rules — complete-case transitions with n < 100 stocks are skipped — so its transition count (~303) is by design lower than this validation CSR's 327.*

---

## Shared Infrastructure

- Signal calendar: **330 dates** (1999-01-29 → 2026-06-10)
- ESTU size: mean **2,494**, min **1,791**, max **3,045**

---

## Known Deviations from PDF Spec

Mirrors the complete NOT-IN-PDF master list from `country_spec.ipynb`.

| # | Decision | Chosen solution | Revisit when |
|---|---|---|---|
| 1 | US-only universe | Single country factor = market intercept | If expanding to multi-country model |
| 2 | Country exposure = 1.0 for all | Implemented exactly; no variation | Never for single-country |
| 3 | Industry constraint implementation | Exact reparametrisation: CW mean of industry exposures ≡ 0 per date | If model structure changes to multi-country |
| 4 | WLS weights | √mcap for ESTU; 0 for non-ESTU stocks | Standard USE4 WLS convention |
| 5 | RF-lag handling | 2 dates skipped (2026-04-30, 2026-05-29): no daily returns available beyond signal_date yet | Will resolve automatically with calendar progression |
| 6 | Warm-up period | 9 factor-date rows dropped (early-1999 dates with <1,000 ESTU stocks) | If warm-up causes coverage gaps post-2001 |
| 7 | CSR returns | Stored per (signal_date, factor) for all 68 factors at each date | For diagnostic use only; not a deliverable |
| 8 | corr(f_c, benchmark) threshold | ≥0.90; observed 0.9987 | If correlation drops below 0.90 |
| 9 | Industry zero-sum check | Max |Σ w_j · f_j| = 3.58×10⁻¹⁸ | If floating point accumulates beyond 10⁻⁶ |

---

## Stage-by-Stage Audit

### Stage 1 — Anchor construction

**PDF SPEC:** Country factor exposure = 1 for all stocks; √mcap WLS weights for ESTU.

**Observed:**
- Anchor rows: **822,918**; **330** dates; **9.5 MB**
- w_reg = √mcap for 822,918 ESTU rows; 0 for non-ESTU

**Status:** ✅ **MATCHES SPEC**

### Stage 2 — Load industry and style factor exposures

**Observed:**
- Industry factors: **55** (from `03_industry_factors/industries_audit`)
- Style factors: **12** (from `02_style_factors/*/`)
- Total factor columns: **68** (1 country + 55 industry + 12 style)

**Status:** ✅ **MATCHES SPEC**

### Stage 3 — Industry zero-sum constraint

**PDF SPEC:** Cap-weighted sum of industry exposures = 0 per date (exact reparametrisation).

**Observed:**
- Max |Σ w_j f_j| (industry, all dates): **3.58×10⁻¹⁸** (target <10⁻⁶)

**Status:** ✅ **MATCHES SPEC**

### Stage 4 — Cross-sectional regression (CSR)

**Implementation:** Per signal_date: WLS of next-period excess log returns on 68 factor exposures (country + 55 industry + 12 style); returns `f_c` (country factor return) and full 68-factor return vector.

**Observed:**
- Transitions run: **327** (2 skipped: 2026-04-30, 2026-05-29 — RF-lag tail)
- Factor-date rows dropped (warm-up): **9** (early 1999)
- CSR returns artifact: **22,236** rows (327 × 68 factors)

**Status:** ✅ **MATCHES SPEC**

### Stage 5 — Validation metrics

**Observed:**
- corr(f_c, benchmark daily return): **0.9987** (target ≥0.90)
- Mean R² of CSR: **0.232**; p10 **0.134**; median **0.211**; p90 **0.356**

**Status:** ✅ **MATCHES SPEC**

---

## Validation Checks

All checks passed on the run dated 2026-06-12.

| Check | Description | Observed | Target / Status |
|---|---|---|---|
| 1 | Anchor rows / dates | 822,918 / 330 | 1,773,820-row ESTU → ✅ PASS |
| 2 | w_reg = 0 for non-ESTU | 0 non-ESTU rows with w_reg > 0 | ✅ PASS |
| 3 | Industry zero-sum max \|Σwf\| | 3.58×10⁻¹⁸ | <10⁻⁶ ✅ PASS |
| 4 | CSR transitions run | 327 | 327 (2 skipped) ✅ PASS |
| 5 | Warm-up drops | 9 | ≤20 ✅ PASS |
| 6 | corr(f_c, benchmark) | 0.9987 | ≥0.90 ✅ PASS |
| 7 | Mean CSR R² | 0.232 | [0.10, 0.60] ✅ PASS |
| 8 | CSR artifact rows | 22,236 | 327 × 68 = 22,236 ✅ PASS |
| 9 | Disk read-back | 0 diff | <10⁻¹⁰ ✅ PASS |
| 10 | Schema check | Passed | ✅ PASS |

---

## Spot Check at 2026-05-29 (last completed transition)

> *2026-05-29 is one of the 2 skipped RF-lag dates. Last completed transition is the date immediately preceding — 2026-04-30 is also skipped. Last valid date in CSR artifact is 2026-03-31.*

- Benchmark corr over full sample: **0.9987**
- Median R²: **0.211**

> *No per-date factor return values were printed by the build notebook beyond the summary statistics above.*
