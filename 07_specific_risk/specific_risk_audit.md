# Specific Risk (Δ) — Construction Audit

**USE4-Faithful Implementation — `specific_risk_build.ipynb`**
Audit Date: 2026-06-22 | USE4 Learning Lab — Reference Audit

---

## Purpose and Scope

Audits the **specific-risk** model — the per-stock idiosyncratic volatility σ_n that forms Δ = diag(σ_n²), the second term of the portfolio-risk identity σ_p² = wᵀ X F Xᵀ w + wᵀ Δ w. With the factor covariance F built in `06_fcov`, Δ completes the risk model. The forecast is the full USE4 stack over the coverage universe: **(1)** a time-series estimate (EWMA + Newey–West on daily specific returns, scaled to the monthly horizon), **(2)** a structural model predicting σ from exposures for thin-history names, **(3)** a coverage blend of the two, **(4)** Bayesian shrinkage toward size-decile means, and **(5)** a daily volatility-regime multiplier.

**Scope.** Covers `specific_risk_build.ipynb` as executed on 2026-06-22. The stage runs after `fcov_build.ipynb` (06); only the risk decomposition (`risk_decomp_build.ipynb`, 08) runs later. **The reference build is estimate-free** — complete-case exposures, no imputation of missing style scores. The numeric kernels (EWMA/Newey–West variance, shrinkage, retransformation constant, bias statistic) live as pure functions in a standalone module with a known-answer test script, per the spec's library policy; construction rationale at spec level: `specific_risk_spec.ipynb`. All values verbatim from the executed notebook; the bias-by-decile comparison is produced by the validation battery.

> *Reference numbers: every value in this document comes verbatim from a reference run of 2026-06-22 (daily specific returns through 2026-04-30). This is a later vintage than the 2026-06-11 reference run behind sections 01–04, so upstream row counts quoted here differ slightly from the earlier audits. Your rebuild will differ in row counts and tail statistics with data vintage; distribution shapes, calibration statistics, and check outcomes should match. Treat persistent sign or order-of-magnitude differences as bugs in your build until proven otherwise.*

---

## Output Schema

**Deliverable:** `data/out/specific_risk.parquet`, written with `compression="zstd"`, `statistics=True`.

| Column | Dtype | Description |
|---|---|---|
| `permaticker` | Int64 | Sharadar permanent ticker ID |
| `signal_date` | Date | Forecast (month-end) date t |
| `in_estu` | Bool | ESTU (regression universe) vs non-ESTU coverage stock |
| `sigma_ts` | Float64 | Layer 1 time-series forecast (null if no daily history) |
| `gamma` | Float64 | Coverage coefficient ∈ [0, 1] — trust in `sigma_ts` |
| `sigma_str` | Float64 | Layer 2 structural forecast (from exposures) |
| `sigma_blend` | Float64 | Layer 3 blend γ·σ_ts + (1−γ)·σ_str |
| `sigma_sh` | Float64 | Layer 4 shrunk (toward size-decile mean) |
| `sigma_spec` | Float64 | **The deliverable** — Layer 5 regime-scaled σ_n (monthly) |
| `mcap` | Float64 | Market cap on `signal_date` |
| `size_decile` | Int8 | Per-date mcap decile (0 small → 9 large) |

The intermediate columns are retained so each layer is auditable; Δ_t = diag(`sigma_spec`²) over the coverage stocks at t. One row per coverage-universe stock-date (post warm-up).

---

## Inputs and Position

- **Daily specific returns (Layer 0)**: `data/out/csr_specific_returns_daily.parquet` — **13,333,028** daily ESTU regression residuals, **6,479** tickers, **6,330** days (2001-02-01 → 2026-04-30), emitted by `daily_csr_build.ipynb` (05).
- **Coverage exposures**: `data/out/industries_use4.parquet` (`in_estu`, `mcap`) + the 12 `data/out/<style>_use4.parquet` scores, complete-case — **1,101,903** stock-dates (ESTU **640,373** + non-ESTU **461,530**).
- **Realized monthly specific returns**: `data/out/csr_specific_returns.parquet` — the out-of-sample validation target (the daily file feeds the regime statistic).
- **Position**: runs after `fcov` (06), before the risk decomposition (08), though its inputs come from 05 and the exposure stack. Consumes the daily residuals and the exposure stack; produces Δ, the specific-risk module's deliverable.

---

## The Five-Layer Construction

**Layer 1 — time series.** Per stock, EWMA variance of daily specific returns (half-life **84**d, mean-zero so no mean subtraction) with a Newey–West Bartlett correction (**5** lags) for the serial correlation; scaled ×21 to the monthly horizon. The NW long-run variance is bounded to [0.25, 4]×V₀ (specific returns carry bid-ask bounce, whose negative autocovariance can collapse a scalar NW estimate) and the monthly vol floored at **1%** (below that is a quiet-window artifact, not a riskless stock). **730,137** stock-dates earn a σ_ts; only **22** hit the floor.

**Layer 2 — structural.** Each date, WLS-regress log σ_ts on the 12 style exposures over the good-history names (γ ≥ 0.9; median **2,028** names/date), √mcap-weighted, and predict σ_str = E₀·exp(x_nᵀ b̂) for *every* coverage stock. E₀ corrects the lognormal retransformation bias by matching the cap-weighted time-series level.

**Layer 3 — blend.** σ_blend = γ·σ_ts + (1−γ)·σ_str, with γ a smooth coverage coefficient (depth × recency). Thin / non-ESTU names (γ → 0) fall back to the structural forecast — **448,952** pure-structural rows.

**Layer 4 — shrinkage.** Empirical-Bayes shrink toward the cap-weighted mean of the stock's size decile: σ_sh = v·σ̄_S(n) + (1−v)·σ_blend, with v = q·|σ−σ̄| / (Δ_S(n) + q·|σ−σ̄|), q = 1. Mean intensity v = **0.354**.

**Layer 5 — vol-regime.** A daily, causal cross-sectional bias statistic B_t = Σ_n w_n·(u_{n,t}/(σ_sh/√21))² (winsorized at ±10), λ = √(EWMA₄₂(B)); σ_spec = λ·σ_sh. λ mean **1.05**, range [**0.75**, **1.94**] (more than doubles the forecast in a crash).

**PIT discipline.** At each month-end t the forecast uses only data dated ≤ t (daily residuals through t; the structural fit and shrinkage groups formed at t; the regime multiplier accumulated causally). The warm-up (pre-2001, before the daily residuals begin) is dropped, leaving **303** forecast month-ends (2001-02-28 → 2026-04-30).

---

## Run Summary (estimate-free canon)

- Deliverable: **1,100,999** stock-dates (**10,714** tickers, **303** month-ends) — ESTU **640,028** + non-ESTU **460,971**, **42.2 MB**.
- Forecast levels: σ_spec median **0.098**/mo over the coverage universe; **0.078**/mo (≈ **27%**/yr) over ESTU — the coverage median is higher because small non-ESTU names carry the most idiosyncratic risk.
- Layer medians: σ_ts (γ ≥ 0.9) **0.070**, σ_str **0.086**, σ_blend **0.088**, σ_sh **0.095**.

---

## Validation Results

The bias statistic standardizes the realized monthly specific return u_{n,t→t+1} by the forecast σ_spec,t (strictly out-of-sample); a calibrated forecast gives a cap-weighted bias ≈ 1. The standardized return is **winsorized at ±10** (the regime-stat cap): a cap-weighted RMS is otherwise non-robust to a single catastrophic idiosyncratic move at the warm-up edge — e.g. an ≈ −80% distressed-stock collapse in March 2001 (permaticker 119419), a name with one month of daily history at the first forecast date so its time-series vol was floored, which alone inflated the *time-series-only* decile-6 statistic to 1.37 (37% of that decile from one point). The deliverable was never affected — that name's γ ≈ 0.08 leans the forecast on the structural model.

| # | Check | Observed | Target / Status |
|---|---|---|---|
| 1 | σ_spec positive, finite, complete coverage | **0** bad / **0** missing rows | ✅ PASS |
| 2 | Overall cap-weighted bias statistic | **1.071** | [0.85, 1.15] ✅ PASS |
| 3 | Bias profile flat across size deciles | spread **0.09**, range [**1.02**, **1.11**] | < 0.40 ✅ PASS |
| 4 | Full stack flattens the TS-only slope | small-cap \|b−1\| **0.45 → 0.06** | spread 0.54 → 0.09 ✅ PASS |
| 5 | Sane levels (ESTU specific vol) | **0.078**/mo (≈ **27%**/yr) | [10%, 45%] ✅ PASS |

**The headline (check 4).** A time-series-only forecast under-predicts the small-cap deciles — bias **1.60, 1.51, 1.24** in deciles 0–2 — because short, noisy histories and bid-ask bounce understate true idiosyncratic risk. The structural model and shrinkage flatten the profile onto 1: the full-stack deciles run **1.02, 1.08, 1.09, …, 1.11** (spread **0.09**). This is the specific-risk analogue of the eigenfactor smile, and it is the load-bearing result — the layers earn their place precisely where the time series fails.

*Bias by size decile (small → large), cap-weighted, winsorized at ±10:*

| decile | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |
|---|---|---|---|---|---|---|---|---|---|---|
| full stack | 1.02 | 1.08 | 1.09 | 1.06 | 1.06 | 1.04 | 1.04 | 1.02 | 1.04 | 1.11 |
| TS-only | 1.60 | 1.51 | 1.24 | 1.13 | 1.10 | 1.09 | 1.08 | 1.07 | 1.07 | 1.05 |

**Honest caveat.** The overall bias sits at **1.071** — a slight, in-band *under*-forecast. The lever to centre it is the EWMA half-life or a small global calibration; both are trivially tunable once the bias battery is wired into your validation contract. The hard part — the flat decile profile — is in hand.

---

## Known Deviations from PDF Spec

USE4 specifies the *structure* (time series + structural + shrinkage + vol-regime) but leaves the parametrization to the implementer. The decisions:

| # | Decision | Chosen solution | Revisit when |
|---|---|---|---|
| 1 | Daily decay basis | Obs-sequence half-life (polars EWMA), not calendar-trading-day age | If thin/gappy names enter the TS set — ESTU is liquid, so the two coincide |
| 2 | NW collapse on bounce | Bound the long-run variance to [0.25, 4]×V₀ | If a horizon other than monthly changes the bounce/signal balance |
| 3 | Minimum specific vol | Floor at **1%**/mo (≈ 3.5%/yr); 17 rows bind | If a genuinely lower-risk universe appears |
| 4 | Coverage coefficient γ | Depth (min(1, obs/252)) × recency (full within 35d, fading to 0 by 95d) | If the blended profile shows a discontinuity in history length |
| 5 | Structural regressors | The 12 style scores + intercept, √mcap-weighted | If industry adds explanatory power, or a style proves perverse |
| 6 | Shrinkage groups / q | Size deciles; q = 1 | Calibrate q against the by-decile bias if the small-cap tail drifts |
| 7 | Vol-regime cadence | Daily bias stat, winsorized ±10, EWMA half-life 42d | If a monthly regime stat suffices, or crashes are missed |
| 8 | Bias-stat winsorization | Standardized return clipped at ±10 (same cap) so one tail event cannot dominate a cap-weighted RMS | If a less / more robust validation metric is wanted |
| 9 | Warm-up | Drop month-ends before any structural fit (pre-2001) | Self-heals; the daily residuals define the start |

---

## Companion Documents

Spec: `07_specific_risk/specific_risk_spec.ipynb`. Kernels: your pure-function module and its known-answer test script, written per the spec's library policy before wiring the build. The standalone validation and the bias-by-decile figure are owned by the spec's validation contract. The daily residuals (Layer 0) are produced by `daily_csr_build.ipynb` (spec: `05_csr/daily_csr_spec.ipynb`); the CSR audit `05_csr/csr_audit.md` covers the monthly build only — the daily CSR is not yet separately audited.
