# Portfolio Risk Decomposition — Construction Audit

**USE4-Faithful Implementation — `risk_decomp_build.ipynb`**
Audit Date: 2026-06-22 | USE4 Learning Lab — Reference Audit

---

## Purpose and Scope

The risk model's **payoff and first end-to-end test**. The exposures X (68 factors), the factor covariance F (built in `06_fcov`) and the specific diagonal Δ (built in `07_specific_risk`) had only ever been validated in isolation; this stage composes them into portfolio risk and the standard Barra/USE4 attribution:

> σ²ₚ = wᵀ X F Xᵀ w (factor) + wᵀ Δ w (specific),  Δ = diag(σ²_spec).

It ships, per rebalance date, (1) the **asset total risk** — every coverage stock's predicted volatility, the diagonal of X F Xᵀ + Δ — and (2) the full **decomposition** of a canonical test-portfolio set: total / factor / specific risk, active tracking error versus the cap-weighted ESTU market, and factor-group risk contributions. The pure decomposition kernels (your kernel module) compute the attribution of *any* portfolio on demand.

**Scope.** Covers `risk_decomp_build.ipynb` as executed on 2026-06-22 — this stage runs **last**, after `specific_risk_build.ipynb`. Estimate-free (complete-case) canon. Spec: `risk_decomp_spec.ipynb`; the kernels are unit-tested by your known-answer test script; standalone results below are from the validation battery, which the spec's validation contract now owns.

> *Reference numbers: every value in this document comes verbatim from a reference run of 2026-06-22 (common month-end dates through 2026-03-31, where the F ∩ Δ intersection binds). This is a later vintage than the 2026-06-11 reference run behind sections 01–04, so upstream row counts quoted here differ slightly from the earlier audits. Your rebuild will differ in row counts and tail statistics with data vintage; distribution shapes, calibration statistics, and check outcomes should match. Treat persistent sign or order-of-magnitude differences as bugs in your build until proven otherwise.*

---

## Output Schemas

**Asset total risk** — `data/out/asset_total_risk.parquet`:

| Column | Dtype | Description |
|---|---|---|
| `permaticker` | Int64 | Sharadar permanent ticker ID |
| `signal_date` | Date | Forecast (month-end) date t |
| `in_estu` | Bool | ESTU vs non-ESTU coverage stock |
| `factor_risk` | Float64 | √(xᵢᵀ F xᵢ) — the stock's systematic vol |
| `specific_risk` | Float64 | σ_spec,ᵢ |
| `total_risk` | Float64 | √(factor² + specific²) — predicted total vol (monthly) |

**Portfolio decomposition** — `data/out/portfolio_risk_decomp.parquet`, per (`signal_date`, `portfolio`): `total_risk`, `factor_risk`, `specific_risk`, `active_te` (tracking error vs the market), `ctr_country` / `ctr_industry` / `ctr_style` / `ctr_specific` (the factor-group risk contributions — they sum to σₚ), and `realized_ret` (the realized portfolio return over (t, t+1], for the bias statistic).

---

## The Decomposition (standard Barra)

For a portfolio w with factor exposures x = Xᵀw:

- **Total risk.** σₚ = √(xᵀ F x + Σᵢ wᵢ² σ²_spec,ᵢ); the factor / specific split is reported directly.
- **Asset contributions.** By Euler's theorem (σₚ is homogeneous degree 1 in w): MCTRᵢ = (X F x + Δw)ᵢ / σₚ, CTRᵢ = wᵢ · MCTRᵢ, and Σᵢ CTRᵢ = σₚ exactly.
- **Factor contributions.** CTRₖ = xₖ (F x)ₖ / σₚ (the Barra "exposure × vol × correlation" form); Σₖ CTRₖ + σ²_specific/σₚ = σₚ. Grouped into country / industry / style.
- **Active risk.** w_active = wₚ − w_b with b = cap-weighted ESTU (the model's market, corr(f_c, market) ≈ **0.998**); the identical formulas give the tracking error and its attribution.

The full 68-column exposure matrix X = [**1** | D | S] (country intercept, 55 industry one-hot dummies, 12 style scores) is assembled directly — *not* the CSR's constraint-reparametrized 56-column design. The dense N×N asset covariance is never formed; everything stays as X F Xᵀ + Δ.

---

## Run Summary (estimate-free canon)

- Universe: **1,044,527** stock-dates over **288** common month-ends (2002-04-30 → 2026-03-31; F is the binding warm-up). **28** UNASSIGNED-industry stock-dates dropped.
- Asset total risk: median **43%**/yr over the coverage universe (**36%**/yr over ESTU — the coverage median is higher because small non-ESTU names carry the most idiosyncratic risk).
- Latest-date (2026-03-31) decomposition, annualized:

| portfolio | σₚ | factor | specific | TE | country | ind | style | spec |
|---|---:|---:|---:|---:|:---:|:---:|:---:|:---:|
| market (cap-wtd ESTU) | **16.6%** | 16.4% | 2.6% | 0.0% | 96% | 2% | 0% | 3% |
| equal-weight ESTU | **21.0%** | 21.0% | 0.8% | 12.9% | 62% | 3% | 34% | 0% |
| momentum long-short | **21.2%** | 21.0% | 2.3% | 23.3% | 5% | 17% | 77% | 1% |
| value long-short | **17.6%** | 17.5% | 2.0% | 31.4% | 37% | 4% | 58% | 1% |
| size long-short | **30.3%** | 30.2% | 2.6% | 23.4% | 39% | 1% | 59% | 1% |

The model attributes risk where it belongs: the **market is the country factor** (96% country, ≈ 16%/yr), a **momentum long-short is momentum-style risk** (77% style, market-neutral), and each tilt's risk is dominated by its own style.

---

## Validation Results

From the validation battery; these checks are owned by the validation contract in `risk_decomp_spec.ipynb`.

| # | Check | Observed | Status |
|---|---|---|---|
| 1 | Asset total risk positive/finite; factor² + specific² = total² | resid **3.3×10⁻¹⁶** | ✅ PASS |
| 2 | Euler additivity (asset CTR & group CTR sum to σₚ) | **1.4×10⁻¹⁷** | ✅ PASS |
| 3 | Cap-weighted ESTU market ≈ country factor | σ **14.4%**/yr; factor **99%**; country CTR **99%** | ✅ PASS |
| 4 | Portfolio bias statistics ~ 1 (predicted vs realized) | market **0.908** | ✅ PASS |
| 5 | Each style tilt's risk is style-dominated | mom **81%**, value **55%**, size **52%** | ✅ PASS |

**End-to-end calibration (check 4).** Standardizing each test portfolio's realized monthly return by its predicted σₚ (winsorized ±10) gives biases **market 0.91**, equal-weight **0.94**, value **0.86**, momentum **0.78**, size **0.80** — all in band, a mild *over*-prediction inherited from the factor model's ≈ 0.95 calibration (the dollar-neutral tilts are almost pure factor bets, so they carry that bias most). This is the first test that X, F and Δ *compose* into a correct portfolio risk forecast.

---

## Known Deviations / Decisions

This stage *uses* the model; the decomposition math is standard Barra. The implementation calls (NOT-IN-PDF decisions):

| # | Decision | Chosen solution | Revisit when |
|---|---|---|---|
| 1 | Exposure form | Full 68-col X (country + 55 one-hot industries + 12 styles), not the CSR's reparametrized 56-col design | Never — the decomposition needs every factor's loading |
| 2 | Benchmark for active risk | Cap-weighted ESTU (the model's market) | If a published index benchmark is required |
| 3 | Coverage policy | Drop UNASSIGNED-industry stocks (28 stock-dates); a portfolio name outside coverage has no risk row | If the uncovered weight in a real book grows material |
| 4 | Common dates | F ∩ Δ = 2002-04-30 → 2026-03-31 (F's warm-up binds) | Self-heals as F's history extends |
| 5 | Test portfolios | Cap-wtd ESTU, equal-weight, three dollar-neutral style long-shorts (mom / value / size) | Add a real holdings book when one is supplied |
| 6 | Bias-stat winsorization | Standardized return clipped at ±10 (same cap as the model's internal stats) | If a less / more robust validation metric is wanted |

---

## Companion Documents

Spec: `08_risk_decomp/risk_decomp_spec.ipynb`. Kernels: your kernel module, validated by your known-answer test script; the standalone validation battery is folded into the spec's validation contract. The inputs: `06_fcov/fcov_audit.md` (F), `07_specific_risk/specific_risk_audit.md` (Δ), `05_csr/csr_audit.md` (the exposures / factor returns).
