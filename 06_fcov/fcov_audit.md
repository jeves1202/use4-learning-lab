# Factor Covariance Forecast (FCOV) — Construction Audit

**USE4-Faithful Implementation — `fcov_build.ipynb`**
Audit Date: 2026-06-16 | USE4 Learning Lab — Reference Audit

---

## Purpose and Scope

Audits the **factor-covariance forecast** — the forward-looking 68×68 matrix F that drives portfolio factor risk, σ_p² = wᵀXFXᵀw + wᵀΔw (this stage delivers F; the specific-risk diagonal Δ is separate work). The build replicates the USE4 covariance pipeline (Menchero, Orr & Wang 2011, §4): a layered sequence — **EWMA base** → **Newey-West** serial-correlation + horizon scaling → **eigenfactor** eigenvalue de-biasing → **volatility-regime** scaling — validated by bias statistics.

**Daily foundation.** The pipeline is a daily machine (Newey-West corrects daily autocorrelation; the eigenfactor de-biasing needs many observations). The shipped CSR is monthly (303 obs, T/N ≈ 4.5), so the covariance is built on a **daily-frequency CSR** (`05_csr/daily_csr_build.ipynb`, the daily sibling of the monthly CSR, specced in `05_csr/daily_csr_spec.ipynb`) — the same constrained-WLS engine, one regression per trading day with month-end exposures held fixed — giving **6,307** daily factor-return days (T/N ≈ 93).

**Scope.** Covers `fcov_build.ipynb` and `daily_csr_build.ipynb` as executed 2026-06-16, the estimator kernel module (unit-tested by its known-answer test script), and the validation battery (owned by the spec's validation contract). The daily CSR is not yet separately audited; its Layer 0 reconciliation is checked here. Complete-case ("estimate-free") run of record. Construction rationale: `fcov_spec.ipynb`.

> *Reference numbers: every value in this document comes verbatim from a reference run of 2026-06-16 (data through 2026-04-30). This is a later vintage than the 2026-06-11 reference run behind sections 01–04, and slightly earlier than the 2026-06-22 run behind the section 05/07/08 audits, so upstream row counts quoted here differ slightly from those audits. Your rebuild will differ in row counts and tail statistics with data vintage; distribution shapes, calibration statistics, and check outcomes should match. Treat persistent sign or order-of-magnitude differences as bugs in your build until proven otherwise.*

---

## Output Schema

All `data/out/*.parquet`, zstd, statistics=True.

- `csr_factor_returns_daily.parquet` — daily factor returns (`date`, `factor`, `factor_type`, `f`, `r2`, `n_stocks`); **430,440** rows over **6,330** days. (Layer-0 input, produced by `05_csr/daily_csr_build.ipynb`.)
- `fcov_factor_cov.parquet` — the final forecast F (stacked upper triangle: `est_date`, `factor_i`, `factor_j`, `cov`); one 68×68 symmetric PSD matrix per month-end. **675,648** rows.
- `fcov_factor_cov_preeig.parquet` — F after Newey-West + horizon, *before* eigenfactor/VRA (the reference for the smile comparison).
- `fcov_predicted_vol.parquet` — `est_date`, `factor`, `vol` (√diag F, monthly horizon); **19,584** rows.

The current forecast is the latest `est_date`. Covariance math lives in your kernel module (pure functions, no I/O); `fcov_build.ipynb` orchestrates it PIT.

---

## The Pipeline

Per month-end t, using daily factor returns dated ≤ t only (PIT):

- **Layer 1 — EWMA base.** EWMA second moments with *split half-lives*: volatilities from the short half-life (84d), correlations from the long (252d), recombined F⁰_ij = R_ij·σ_i·σ_j.
- **Layer 2 — Newey-West.** Bartlett-weighted HAC correction V₀ + Σ_{l=1..L} (1 − l/(L+1))(V_l + V_lᵀ), L = 5 (PSD by construction), then scaled daily→monthly (×21).
- **Layer 3 — Eigenfactor.** Monte-Carlo eigenvalue de-biasing (1,000 sims): the optimizer-critical step (see "The Eigenfactor Result" below).
- **Layer 4 — Volatility regime.** F ← λ_vol²·F, with λ_vol = √EWMA(B_t) on the daily cross-sectional bias B_t = (1/68)·Σ_k (f_{k,t}/σ_{k,t})² (half-life 42d).

**Run.** Daily panel **6,307** days × 68 (2001-04-02 → 2026-04-30); **288** monthly forecasts (2002-04-30 → 2026-03-31, after a 252-day warm-up). VRA multiplier mean **1.025** (min 0.69, max 2.22 — it scales up in crises). Latest forecast 2026-03-31: country monthly vol **4.7%** (≈16% annualized); F PSD (min eigenvalue 3.7×10⁻⁵).

---

## Validation Results

From the validation battery (288 monthly out-of-sample forecasts); these checks are owned by the validation contract in `fcov_spec.ipynb`.

| # | Check | Observed | Target |
|---|---|---|---|
| 1 | Every forecast 68×68 symmetric & PSD | min eigenvalue **3.9×10⁻⁶**; asym **0** | PSD ✅ PASS |
| 2 | Per-factor bias statistic ≈ 1 | mean **0.949**; country **0.911**; median **0.966** | [0.85, 1.15] ✅ PASS |
| 3 | Eigenfactor de-biasing flattens the smile | mean\|b−1\| **0.310 → 0.074**; low-10 **0.61 → 0.11**; **97%** within 2·CI | improves ✅ PASS |
| 4 | Sane predicted vols | median country monthly vol **4.14%**; all variances > 0 | [2, 8]% ✅ PASS |
| 5 | OOS robustness: calibration stable across eras | early (n=144) bias **0.941**, eigenfactor \|b−1\| **0.31 → 0.10**; late (n=144) bias **0.949**, **0.31 → 0.06** | each half holds ✅ PASS |

**Out-of-sample robustness.** Nothing in the model is fit — the half-lives and the eigenfactor scaling a = 1.2 are fixed priors, and every forecast is strictly PIT — so the backtest is already out-of-sample by construction. Splitting it at the median date confirms the calibration is not a full-sample average artifact: the per-factor bias is essentially identical in both halves (**0.941** vs **0.949**), and the eigenfactor de-biasing flattens the smile equally in each disjoint 12-year era (**0.31 → 0.10** early, **0.31 → 0.06** late). The mild conservatism is structural and stable, not luck; the priors generalize without tuning.

The estimator kernels are additionally unit-tested on synthetic data by your known-answer test script: EWMA recovers a known Σ; Newey-West lifts an AR(1) series' variance toward its long-run value; the eigenfactor bias curve has v(small) > 1 > v(large) and the adjusted F stays PSD.

**Layer 0 reconciliation.** The daily CSR reconciles to the monthly CSR: summing daily factor returns over each month tracks the monthly factor return at corr **0.9913** (RMSE 0.47pp); the daily country factor tracks the daily ESTU market at corr **0.9989**; condition number 49.9 every day.

---

## The Eigenfactor Result (the headline)

The sample covariance systematically **underpredicts** the risk of its lowest-variance eigenportfolios — the bias that drives optimizers to over-allocate to apparently-low-risk positions. The backtest shows it starkly: *before* de-biasing, the lowest-variance eigenfactors realize up to **1.95×** their predicted risk (bias statistics **1.4–2.0** across the low end), while the highest-variance ones are mildly overpredicted — the characteristic downward "smile."

The Monte-Carlo eigenvalue adjustment (Menchero, Wang & Orr 2011) corrects it: the mean absolute bias deviation collapses from **0.310** to **0.074** (a 4× flattening), the low-10 eigenfactors from **0.61** to **0.11**, and **97%** of eigenfactors land within 2·CI of 1. The before/after smile and the per-factor bias bars are rendered as a figure by the validation battery. This is the most important correction for a model used in optimization, and it validates cleanly.

---

## Known Deviations from PDF Spec

USE4 specifies the pipeline structure, not every constant; the spec §9 (`fcov_spec.ipynb`) carries the full master list. Headline calls:

| # | Decision | Chosen solution | Revisit when |
|---|---|---|---|
| 1 | Daily vs monthly returns | Daily CSR (Layer 0) — USE4 is a daily machine | Never — monthly guts Newey-West & the eigenfactor de-biasing |
| 2 | EWMA half-lives | 84d vol / 252d corr (split; vol faster) | Retune against the per-factor bias battery |
| 3 | Newey-West lags | 5 daily Bartlett lags | If the autocorrelation profile changes |
| 4 | Eigenfactor T_win / a | EWMA effective obs; a = 1.2; equal-weight sim | Tune a so the eigenfactor bias sits inside the CI |
| 5 | VRA half-life | 42d causal EWMA of the cross-sectional bias | If regime tracking lags or over-reacts |
| 6 | Complete-panel window | start at first all-68-present day; rare later gaps filled 0 | If an industry goes persistently absent |

**Out of scope (documented alternatives):** GARCH-family (DCC, GJR/EGARCH) and Ledoit-Wolf shrinkage — at 68×68 the factor model is already the dimension reduction, so the eigenfactor adjustment plays the bias-correction role shrinkage would at the asset level. Specific risk Δ is a separate deliverable.

---

## Companion Documents

Spec: `06_fcov/fcov_spec.ipynb` (construction rationale; owns the validation contract). Library + tests: your estimator kernel module and its known-answer test script, as specified there. Validation: the validation battery folded into `fcov_spec.ipynb`'s validation contract (the input covariance-structure checks live in `05_csr/csr_spec.ipynb`'s validation contract). Upstream: `05_csr/csr_audit.md` (factor returns), `04_country_factor/country_audit.md`.
