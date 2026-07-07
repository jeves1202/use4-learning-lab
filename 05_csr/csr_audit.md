# Cross-Sectional Regression (CSR) — Construction Audit

**USE4-Faithful Implementation — `csr_build.ipynb`**
Audit Date: 2026-06-22 | USE4 Learning Lab — Reference Audit

---

## Purpose and Scope

Audits the **production cross-sectional regression** — the step that turns the full USE4 exposure stack into the model's two return deliverables. Each month it regresses realized stock returns on X = [**1** | D | S] (country, 55 industry dummies, 12 standardized styles), √mcap-weighted, under the cap-weighted industry constraint Σ_j w_jt · f_ind,j,t = 0 (USE4 Methodology Notes §2.1; Empirical Notes §4), and ships the **factor returns** (the 68 series) and the **specific (idiosyncratic) returns** (the per-stock residuals over the whole coverage universe — ESTU in-sample plus non-ESTU out-of-sample — that feed the specific-risk model).

**Single canonical source.** The constrained-WLS engine first appears in section 04 as the country build's self-check (the *validation CSR*, `data/out/csr_validation_returns.parquet`); the production version lives here. This stage is the sole producer of the model's factor returns and adds the specific-return deliverable; once the production returns reconcile against the validation CSR, that artifact is retired and the country build is anchor-only (see *Single Canonical Source* below). The country-factor validation the parallel CSR used to provide — corr(f_c, market) ≈ 0.999 — is one of this stage's checks (*Validation Results*).

**Scope and run mode.** Covers `csr_build.ipynb` as executed on 2026-06-22 (data through 2026-06-18), run after the country build (04). **The reference run is estimate-free** (complete-case): stocks missing any of the 12 style scores are dropped, not imputed. All values are verbatim from the executed notebook; it generates no figures. Construction rationale at spec level: `05_csr/csr_spec.ipynb` (§4, the constraint algebra and the full engine); the country-factor identification it implements is derived in `04_country_factor/country_spec.ipynb`.

**Monthly only.** This audit covers the monthly production CSR (`csr_build.ipynb`) only. The daily sibling (`daily_csr_spec.ipynb`, built by `daily_csr_build.ipynb`) is not yet separately audited; this will be revisited when the audits are brought up to date.

> *Reference numbers: every value in this document comes verbatim from a reference run of 2026-06-22 (data through 2026-06-18). This is a later vintage than the 2026-06-11 reference run behind sections 01–04, so upstream row counts quoted here differ slightly from the earlier audits. Your rebuild will differ in row counts and tail statistics with data vintage; distribution shapes, calibration statistics, and check outcomes should match. Treat persistent sign or order-of-magnitude differences as bugs in your build until proven otherwise.*

---

## Output Schema

Two deliverables, both under `data/out/`, written `compression="zstd"`, `statistics=True`.

### `csr_factor_returns.parquet` — factor returns

| Column | Dtype | Description |
|---|---|---|
| `signal_date` | Date | Exposure date t (start of the return month) |
| `ret_date` | Date | End of the return month (the next signal date) |
| `factor` | String | `country`, the 55 industry names, or the 12 style score names |
| `factor_type` | String | `country` / `industry` / `style` |
| `f` | Float64 | Factor return for the month (null if the column was degenerate that month) |
| `r2` | Float64 | Weighted cross-sectional R² of the month's regression (repeated per row) |
| `n_stocks` | UInt32 | Stocks in the month's regression (repeated per row) |

### `csr_specific_returns.parquet` — specific (idiosyncratic) returns

| Column | Dtype | Description |
|---|---|---|
| `permaticker` | Int64 | Sharadar permanent ticker ID |
| `signal_date` | Date | Exposure date t |
| `ret_date` | Date | End of the return month |
| `in_estu` | Bool | `True` = ESTU regression residual (in-sample); `False` = non-ESTU coverage stock (out-of-sample, fitted f applied) |
| `y` | Float64 | Realized excess log return over (t, t+1] |
| `y_hat` | Float64 | Factor-explained return X f |
| `u` | Float64 | Specific return u = y − ŷ |
| `w_reg` | Float64 | √mcap_i / Σ_{j∈ESTU realized} √mcap_j — one scale for both groups (ESTU sums to 1 per date) |

68 factor rows per solved transition (1 + 55 + 12); one specific-return row per coverage-universe stock-date (ESTU in-sample + non-ESTU out-of-sample, flagged `in_estu`).

---

## Inputs and Position

- **Anchor**: `data/out/country_use4.parquet` — **822,937** in-ESTU rows, **330** dates (1999-01-29 → 2026-06-18), country exposure ≡ 1, Σ `w_reg` = 1 per date (max deviation **1.1×10⁻¹⁶**).
- **Exposures**: the 12 `<style>_use4.parquet` score columns + `industries_use4.parquet` (the coverage universe: `in_estu`, `mcap`, `industry`) + `industry_weights_use4.parquet` (55 industries, the constraint weights).
- **Returns**: the shared daily panel built in `01.5_daily` (`data/out/daily_returns.parquet`, **39,021,853** rows) compounded to **1,778,631** monthly excess log stock-returns and a **329**-month benchmark.
- **Position**: runs immediately after the country build — `04 country → 05 csr_build → 05 daily_csr_build → 06 fcov_build → 07 specific_risk_build → 08 risk_decomp_build`. Reads the country anchor and every upstream exposure deliverable; owns the regression engine and the specific-return output (ESTU + non-ESTU coverage).

---

## The Regression Engine

Per month-transition (t, t+1] the build forms X = [**1** | D | S], weights by √v (v = √mcap normalized), eliminates the **largest-cap-weight present industry** e (the numerically safest divisor), solves the reduced system [**1** | D̃ | S] by √v-scaled least squares, and recovers f_e = −(Σ_{j≠e} w_j f_j)/w_e from the constraint. This is **exact restricted least squares** — the choice of e changes conditioning, nothing else. This is the model's single production CSR engine; the validation CSR in 04 is the same engine run as a self-check and retires once reconciled (see *Single Canonical Source* below).

**Specific returns — the deliverable.** After f is recovered, the build forms ŷ_i = X_i f and the specific return u_i = y_i − ŷ_i for every regression (ESTU) stock, and ships them with y, ŷ, u, w_reg. Two identities hold by construction and are asserted: y = ŷ + u exactly, and Σ_i v_i u_i = 0 per date over ESTU (the WLS residual is orthogonal to the country intercept — the cap-weighted ESTU specific return is zero).

**Coverage extension.** The specific-risk module forecasts the whole coverage universe, not just ESTU, so the same fitted f is applied *out-of-sample* to the non-ESTU coverage stocks: ŷ_i = f_c + f_ind[ind_i] + S_i · f_style, u_i = y_i − ŷ_i — no re-estimation. Non-ESTU rows are complete-case (all 12 styles finite, mcap > 0, a realized return, an ESTU-present industry; UNASSIGNED dropped) and flagged `in_estu` = `False`. Being genuinely out-of-sample they carry no orthogonality guarantee and run wider than the ESTU residuals.

---

## Run Summary (estimate-free)

- Transitions solved: **303** of 329 attempted; skipped **2** at the RF-publication tail (no excess returns) and **24** early-1999/2000 complete-case-thin (whole style columns uncovered that early in the sample under the complete-case rule).
- Degenerate style factor-dates (f = null): **0**. Industry factor-dates dropped (no members, complete-case): **23**.
- CSR sample sizes: min **100**, median **2,164**, max **2,388**. UNASSIGNED-industry stock-dates dropped: **0**.
- Factor returns: **20,604** rows (country 303, industry 16,665, style 3,636), **0.16 MB**.
- Specific returns: **1,097,079** stock-dates (**10,699** tickers, **32.6 MB**) — ESTU **637,392** (in-sample, u std **0.094**) + non-ESTU **459,687** (out-of-sample, u std **0.175**). The non-ESTU set is complete-case 461,530 less 28 UNASSIGNED and 39 ESTU-absent-industry stock-dates.

---

## Validation Results

All values verbatim from the executed notebook (estimate-free reference run).

| # | Check | Observed | Target |
|---|---|---|---|
| 1 | Specific-return identity y = ŷ + u (all rows) | **0.00×10⁰** | <10⁻¹² ✅ PASS |
| 2 | max \|Σ w_reg·u\| per date, **ESTU** (residual ⊥ country) | **5.98×10⁻¹⁶** | <10⁻⁹ ✅ PASS |
| 3 | corr(f_c, benchmark monthly excess return) | **0.9982** | >0.99 ✅ PASS |
| 4 | max \|Σ_j w_j·f_ind,j\| over dates | **5.39×10⁻¹⁸** | <10⁻¹⁰ ✅ PASS |
| 5 | Max condition number of the √v-scaled design | **49.9** | <10⁶ ✅ PASS |
| 6 | Mean weighted cross-sectional R² | **0.252** (median **0.236**) | >0.15 ✅ PASS |
| 7 | Coverage-flag integrity (no stock-date both ESTU & non-ESTU) | **637,392** + **459,687**; 0 cross | both populated ✅ PASS |
| 8 | Non-ESTU specific-return std (out-of-sample, sane) | **0.175** (ESTU **0.094**) | 0 < std < 1 ✅ PASS |

**Informational.** f_c annualized vol **16.1%**; monthly RMSE(f_c − benchmark) **0.28pp** — the country factor is the market.

---

## Single Canonical Source

When this stage first shipped, the country build (04) emitted a parallel *validation CSR* (`data/out/csr_validation_returns.parquet`) and the production returns reconciled against it to floating-point noise (max |Δf| ≈ 1×10⁻¹⁵ over all **20,604** rows) — the proof the two engines were identical. That transition is now **complete**: the country build is anchor-only, the validation artifact is retired, and `csr_build.ipynb` is the single canonical source of factor returns (and the owner of specific returns). The country-factor validation the parallel CSR used to provide — corr(f_c, market) **0.9982** — is check 3 above.

The two runs differ in transition count by design: 04's validation CSR imposes no minimum-sample skip and solves **327** of 329 transitions (only the RF-lag tail is skipped), while the production CSR here skips the 24 complete-case-thin early transitions (n < 100) as well and solves **303**. The reconciliation covers the transitions the production CSR solves.

---

## Known Deviations from PDF Spec

This stage owns the regression engine; its full NOT-IN-PDF list — constraint implementation, eliminated industry, return convention, degenerate columns, unclassifiable stocks, RF-lag skips, constraint-weight vs realized-sample — is specified in `05_csr/csr_spec.ipynb` §6 (the country-factor identification it implements is derived in `04_country_factor/country_spec.ipynb`). The deviation-history and CSR-deliverable calls:

| # | Decision | Chosen solution | Revisit when |
|---|---|---|---|
| 1 | Specific-return universe (*done*) | Whole coverage universe: ESTU residuals (`in_estu`) + non-ESTU via out-of-sample fitted f, complete-case | — revisit only if the coverage universe is redefined |
| 2 | Single canonical CSR (*done*) | The country build is anchor-only; `csr_build.ipynb` is the sole production CSR. `csr_validation_returns.parquet` retired | — transition complete |
| 3 | `factor_type` column | Tag each row `country`/`industry`/`style` for downstream grouping | Never — one column saves every consumer a lookup |
| 4 | Specific-return columns | Store y, ŷ, u, w_reg (self-validating: y = ŷ + u re-checkable; w_reg for weighted specific variance) | If storage matters — u alone suffices for variance |

---

## Companion Documents

Spec: `05_csr/csr_spec.ipynb` (the regression engine, including the constraint algebra; country-factor identification derived in `04_country_factor/country_spec.ipynb`). Daily sibling: `05_csr/daily_csr_spec.ipynb` — not yet separately audited. Upstream evidence: `04_country_factor/country_audit.md` (the anchor), the per-style `02_style_factors/<slug>/<slug>_audit.md`, and `03_industry_factors/industries_audit.md`.
