# Factor Overview

The 12 USE4-style factors, in a sensible build order. MoM ρ is the
month-over-month rank stability from our reference run (2026-06-11) — your
values should land nearby.

| # | Factor | One-liner | Inputs | Needs | Ref. MoM ρ |
|---|---|---|---|---|---|
| 1 | **size** | ln(market cap), standardized | DAILY/SEP mcap | ESTU | 0.9967 |
| 2 | **beta** | EW-OLS slope on ESTU benchmark, 252d / 63d half-life | SEP returns, RF | ESTU | 0.9458 |
| 3 | **bp** | book equity / market cap | SF1 ARQ equity | ESTU | 0.9773 |
| 4 | **eyld** | 0.6·CETOP + 0.4·ETOP (EPFWD unavailable) | SF1 TTM netinc, depamor | ESTU | 0.9630 |
| 5 | **dyld** | TTM DPS / price, split-adjusted, corrupt-guarded | SF1 ARQ dps, ACTIONS | ESTU | 0.9927 |
| 6 | **lev** | 0.75·MLEV + 0.15·DTOA + 0.10·BLEV (standardized subs) | SF1 ARQ balance sheet | ESTU | 0.9908 |
| 7 | **liq** | 0.35·STOM + 0.35·STOQ + 0.30·STOA (log turnover) | SEP volume, sharesout | ESTU | 0.9706 |
| 8 | **gro** | 0.90·EGRO + 0.10·SGRO (EGRLF proxied) | SF1 annual EPS/SPS | ESTU | 0.9863 |
| 9 | **mom** | weighted mean daily excess return, 504d/21d lag/126d HL | daily panel | ESTU + daily | 0.8702 |
| 10 | **resvol** | 0.75·DASTD + 0.15·CMRA + 0.10·HSIGMA, ortho to Beta | daily panel | beta | 0.9397 |
| 11 | **nlb** | cube of standardized Beta, ortho to Beta | beta_use4 output | beta | 0.8723 |
| 12 | **nls** | cube of standardized Size, ortho to Size | size_use4 output | size | 0.9882 |

Suggested path: **size → beta → bp** (one spot fundamental, one time-series, one
PIT-join factor — you'll touch every mechanic), then extract the **daily panel**,
then the rest in any order respecting `beta → {resvol, nlb}` and `size → nls`.

Two structural facts to internalize early:

- **One ESTU, everywhere.** All standardization moments come from the same
  estimation universe; factor-specific universes silently break the
  multi-factor regression later.
- **The factor-on-factor inputs are deliberate.** nlb consumes your *built*
  beta deliverable, nls your *built* size deliverable — orthogonalizing to a
  locally recomputed exposure only partially removes the collinearity these
  factors exist to remove.

For every forced deviation from the published USE4 definitions (no analyst
estimates, no float, no preferred equity, ...) see
`docs/deviations_register.tex`.

---

## In-depth factor discussions

### 1. Size

**Economic role.** Size captures the persistent return spread between small-cap and large-cap stocks. Empirically, small caps have outperformed large caps over long horizons in most markets — attributed to liquidity premia, limited analyst coverage, and higher idiosyncratic risk. In a risk model, Size serves a second purpose: many other factors (Value, Liquidity, Volatility) are strongly correlated with market cap, so controlling for Size isolates the economically distinct signals in those factors.

**USE4 specification.** SIZE = 1.0 · LNCAP, where LNCAP = ln(total market capitalization). One descriptor, weight 1.0 — the simplest factor in the model. The log transformation compresses the right tail of market-cap distributions, making the cross-sectional dispersion approximately Gaussian and the factor exposures well-behaved for standardization.

**Construction.** At each month-end signal date, take the most recent SEP `marketcap` (in millions of dollars) via a backward asof join, then compute `lncap = ln(marketcap × 1,000,000)`. Standardize cross-sectionally using cap-weighted mean and equal-weighted std from ESTU. Both `lncap` and `size_score` are saved: `lncap` is needed downstream as a WLS regression weight (`exp(lncap/2) = √mcap`) and as the orthogonalization target for NLS.

**Key deviations.** Two choices differ from the generic USE4 trim recipe, both documented in deviations register B8. First, the trim center uses the **equal-weighted mean** rather than the cap-weighted mean: for `lncap`, the cap-weighted mean sits ~4 log units above the equal-weighted mean (large-caps dominate), so a CW-centred 3σ band clips ~20% of small-cap ESTU rows — the wrong tail. Second, the multiplier is **4σ rather than 3σ**: `lncap` is approximately Gaussian (log of a log-normal), so a 4σ bound clips less than 0.1% of ESTU in practice while preserving the full large-cap tail. Separately, the factor uses total market cap rather than float-adjusted market cap because Sharadar provides only total cap; this pervades ESTU weights and the Size exposure alike (deviations register A3).

**Character.** Highest MoM stability of any factor (ρ = 0.9967) — market cap changes gradually except during extreme events. The median ESTU stock has a negative `size_score` because most stocks by count are small; the cap-weighted mean, which equals zero by construction, sits in large-cap territory.

---

### 2. Beta

**Economic role.** Beta measures a stock's sensitivity to market-wide moves — the fraction of return variance explained by the market factor. In USE4 it is the highest-signal style factor by |t|-statistic (average 3.96–4.02 per the Empirical Notes) and the only style factor with non-trivial correlation to the cap-weighted ESTU (0.77–0.90), because tilting toward high-beta stocks is mechanically equivalent to levering the market. Crucially, Beta and the Country factor are distinct: the Country factor describes the common US market return; Beta describes each stock's *sensitivity* to it. A high-beta tilt on a long-only portfolio carries more market risk than a pure market exposure — that extra risk is what Beta prices.

**USE4 specification.** BETA is the OLS slope in `r_it − r_ft = α + β·R_t + e_t`, estimated over 252 trailing trading days with a 63-day exponential half-life. R_t is the cap-weighted excess return of the ESTU. Final factor: BETA = 1.0 · BETA (single descriptor).

**Construction.** The exponential half-life weights recent observations more heavily: a return 63 days ago gets half the weight of today's return. For each stock at each signal date, slice the prior 252 trading days of excess returns, attach exponential weights, and solve the weighted normal equations. Minimum 63 observations; stocks with fewer get null beta. The benchmark uses **lagged market caps** (prior-day `mcap`) to avoid contemporaneous-weight bias in the return. Standardize cross-sectionally.

**Key facts.**
- The cap-weighted mean of raw beta across ESTU ≈ 1.0 by construction — the ESTU portfolio's beta against itself is identically 1.
- Build Beta before NLB and Residual Volatility; both consume `beta_use4.parquet` as an upstream input.
- Beta is not explicitly winsorized in the USE4 spec (unlike NLB/NLS); the general 3σ trim from Methodology Notes §2.2 is applied as a lightweight guard.

**Character.** MoM ρ = 0.9458, moderate stability. The 252-day window makes month-to-month changes gradual, but the exponential half-life keeps Beta more responsive to recent regime shifts than a naive rolling OLS would be.

---

### 3. Book-to-Price (bp)

**Economic role.** Book-to-Price (the inverse of Price/Book) is the canonical value factor: stocks trading cheaply relative to their accounting book value have historically outperformed. The mechanism is contested — mispricing, distress compensation, or mean-reverting investor sentiment — but the empirical spread is robust across markets and decades. In USE4, BP is one of three value-oriented factors alongside Earnings Yield and Dividend Yield; together they form the "value cluster." BP is the purest accounting-based value signal; its complement factors (EYLD, DYLD) blend in cash generation and income signals.

**USE4 specification.** BTOP = book equity / current market cap. Book equity = total shareholders' equity to the parent minus preferred equity. Market cap is contemporaneous (signal-date price, not filing-date price).

**Construction.** Load SF1 with `dimension = "ARQ"` (as-reported quarterly), PIT-keyed on `datekey` with an 18-month staleness window. Book equity = SF1 `equity`. Market cap = `estu_monthly.mcap` at signal date — never SF1's filing-date market cap, which may be months stale by the time the factor is evaluated.

**Forced deviations.**
- **Preferred equity absent** (deviations register A4). USE4 defines book equity as total equity minus preferred equity; SF1 carries no preferred equity balance field. The `equity` field in SF1 is already the common equity figure per GAAP (common stock + APIC + retained earnings + AOCI), so it is conceptually correct for most companies, but the precise preferred equity component cannot be verified or subtracted. The error is concentrated in financials and utilities where preferred stock is most common and damps the factor (slightly overstating BE) rather than inverting it.
- **Negative book equity retained** (deviations register B3). ~49,000 ARQ rows carry negative equity (buy-back-heavy and distressed firms). These are kept as negative BTOP rather than nulled: they should rank below low-BTOP growth names in the value ordering, which negative values accomplish automatically.

**Character.** MoM ρ = 0.9773. Book value is a balance-sheet stock that moves slowly; the main source of month-to-month change is the price denominator, which updates daily but is sampled at signal-date values.

---

### 4. Earnings Yield (eyld)

**Economic role.** Earnings Yield (E/P) is the USE4 Empirical Notes' "most significant factor in explaining value-related return differences in U.S. equities." High E/P stocks trade at low multiples relative to earnings — classically interpreted as undervaluation or compensation for higher distress or cyclicality risk. The USE4 composite blends forward-looking analyst consensus with trailing signals to capture both valuation level and near-term earnings momentum; the forward component dominates at 75% because markets are forward-looking.

**USE4 specification.** EYLD = 0.75 · EPFWD + 0.15 · CETOP + 0.10 · ETOP. EPFWD is 12-month forward EPS / market cap (analyst consensus). CETOP is trailing 12-month cash earnings / price. ETOP is trailing 12-month net earnings / market cap.

**Forced deviation.** EPFWD is entirely absent from Sharadar (no analyst consensus data). Proxying EPFWD with trailing EPS would duplicate ETOP with no added information while pretending to forward content. The honest choice is to drop EPFWD and renormalize: **EYLD = 0.60 · CETOP + 0.40 · ETOP** (preserving the 0.15:0.10 ratio between the remaining descriptors). The result is a pure trailing value measure; what is lost is the forward-looking revision signal. See deviations register A1.

**Construction.** TTM figures are constructed as 4-quarter rolling sums over `calendardate` from the ARQ dimension (the SF1 extract carries ARQ only; the TTM dimension is not available — deviations register A8). Cash earnings = `netinc + depamor`. The market cap denominator is always `estu_monthly.mcap` at signal date — SF1's filing-date market cap is months stale and wrong for an E/P ratio that should reflect current valuation. Negative earnings are allowed; the 3σ trim handles extremes.

**Character.** MoM ρ = 0.9630. Trailing earnings are sticky, but large write-downs and cyclical swings can produce jumps, particularly in energy and financials. ESTU median raw eyld ≈ 0.059 (a realistic US earnings yield), with strictly monotone quintiles and spread 0.260.

---

### 5. Dividend Yield (dyld)

**Economic role.** Dividend Yield reflects the income component of total return and a signaling effect: companies that pay and grow dividends tend to be profitable, mature, and confident in future cash generation. In USE4, Dividend Yield is a value-adjacent factor that differentiates mature cash-generative businesses from high-growth names. Its distinctive characteristic — versus BP and EYLD — is that the majority of the universe (≈63%) pays no dividend, creating a large mass point at zero that the factor construction must handle deliberately.

**USE4 specification.** DTOP = trailing 12-month DPS / current price. Single descriptor.

**Construction.** TTM DPS is the 4-quarter sum of split-adjusted DPS from ARQ, requiring at least 1 complete quarter (so zero-payers survive with `dyld = 0.0`). DPS is rebaselined across split-affected quarters using cumulative split factors from ACTIONS — 217,223 split-affected quarters on the reference run. The denominator is the signal-date price from SEP.

**Key implementation facts.**
- **Zero-payers are zeros, not nulls** (deviations register B4). A company that pays no dividend has `dyld = 0.0` — a real economic statement. Treating zero-payers as missing would shrink coverage by ~63% and turn DYLD into a payer-only factor. Quintiles Q1 and Q2 will have mean exactly 0 (the zero-payer mass); Q3–Q5 strictly increase to 0.056.
- **Corrupt-DPS guard** (deviations register B2). Raw SF1 `dps` contains split-era artefacts implying physically impossible yields. Any quarterly DPS implying a single-quarter yield >50% is nulled at source (1,292 filings on the reference run); the same 50% threshold backstops the TTM yield (8,700 nulled). No legitimate dividend policy produces a 50%-per-quarter yield — these are share-count mismatches, not real dividends.
- **TTM constructed from ARQ** because the extract carries ARQ only (deviations register A8).

**Character.** Highest MoM stability of all fundamental factors (ρ = 0.9927). Dividend policy is extremely sticky; the main source of month-to-month variability is the price denominator.

---

### 6. Leverage (lev)

**Economic role.** Leverage captures the financial risk of a company's capital structure. Highly levered firms have more volatile equity returns — debt service is a fixed cost that magnifies equity swings — and higher distress risk. In a risk model, Leverage loads on the spread between levered and unlevered equity returns independently of the market factor; it isolates structural balance-sheet risk that Beta and Volatility do not fully capture.

**USE4 specification.** LEV = 0.75 · MLEV + 0.15 · DTOA + 0.10 · BLEV, where:
- **MLEV** (market leverage) = (market equity + preferred equity + long-term debt) / market equity
- **DTOA** (debt to assets) = total debt / total assets
- **BLEV** (book leverage) = (book equity + preferred equity + long-term debt) / book equity

MLEV gets the dominant weight because market equity is forward-looking and reflects the current burden of debt relative to going-concern value.

**Construction and deviations.**

The sub-descriptors live on incompatible scales: MLEV and BLEV both floor at 1.0 (unlevered minimum) and range to ~20+; DTOA is bounded in [0, 1]. A naive weighted sum would give MLEV far more than its intended 75% share purely from scale. The solution is to **standardize each sub-descriptor cross-sectionally before compositing** (deviations register B7) — the published weights are only meaningful on a common scale, and USE4's own standardization convention provides it. Post-standardization the CW mean of each sub-descriptor is machine-zero (residual ≤ 6×10⁻¹⁶).

**Preferred equity absent** (deviations register A4). With PE = 0: MLEV = (mcap + debtnc) / mcap; DTOA unchanged; BLEV = (equity + debtnc) / equity. Note the PE terms cancel algebraically in BLEV's numerator (BE + PE = equity in USE4's definition), so only MLEV's numerator and BLEV's denominator are affected. The error damps the factor for financials and utilities — the heaviest preferred-stock issuers still sort to Q5.

**Null debt treated as zero** (deviations register B6). `debtnc` is null in 20.3% of ARQ rows. For balance-sheet debt, absence of a line item means the firm carries none — it is not unknown data. Treating nulls as missing would delete one-fifth of the universe, biased toward exactly the unlevered firms LEV must rank lowest.

**Character.** MoM ρ = 0.9908. Capital structure changes slowly; leverage is highly persistent. The ESTU clip rate is 2.71%, above the 0.5–2% target band (flagged WARN in the audit); this reflects LEV's structurally heavy right tail rather than a parameter error.

---

### 7. Liquidity (liq)

**Economic role.** Liquidity captures the cost of trading a stock. Illiquid stocks command a premium (investors demand compensation for the trading friction they incur when exiting), but they also introduce noise — returns of thinly traded names reflect microstructure effects as much as fundamental information. In a risk model, Liquidity isolates return variation driven by trading friction, independently of Size (with which it is correlated but not identical).

**USE4 specification.** LIQ = 0.35 · STOM + 0.35 · STOQ + 0.30 · STOA, where each descriptor is the **log of share turnover** over a different horizon:
- **STOM** = ln(cumulative volume / shares outstanding) over the trailing month (~21 trading days)
- **STOQ** = same over the trailing quarter (~63 days)
- **STOA** = same over the trailing year (~252 days)

Taking the log of the sum (not the mean) makes the distribution approximately Gaussian and compresses the enormous range between a micro-cap OTC name and a mega-cap during a short squeeze.

**Construction.** Share turnover = cumulative volume over the window / shares outstanding on the signal date. Shares outstanding is derived from `marketcap / closeunadj` per SEP row. The three-horizon blend captures both current trading activity (STOM) and structural liquidity (STOA); a stock that traded heavily last month but is normally illiquid won't score as a liquid stock.

**Key data issue.** 20,710 SEP rows (0.057%, 2,221 tickers) show turnover >1 — physically impossible under normal market conditions. These are vendor share-count artefacts. The log transform plus 3σ trim absorb them without row-level surgery; at 0.057% of rows they are too sparse to move monthly medians. See deviations register A7. The median daily turnover ≈ 0.00461 sits inside the 0.001–0.010 plausibility band.

**Character.** MoM ρ = 0.9706. A stock's trading environment doesn't change dramatically month-to-month. Short-term events (earnings, news) can create spikes in STOM, but STOQ and STOA dampen them — which is exactly what the three-horizon blend is designed to accomplish.

---

### 8. Growth (gro)

**Economic role.** Growth captures the market premium for companies with strong earnings and sales expansion. High-growth firms typically command high multiples; the factor isolates whether that growth premium is justified by the fundamental trajectory. In USE4, Growth is negatively correlated with Earnings Yield and Book-to-Price — high-growth firms tend to have high P/E and P/B, so growth exposure is the complement of the value cluster. It is the factor most sensitive to the missing analyst data.

**USE4 specification.** GRO = 0.70 · EGRLF + 0.20 · EGRO + 0.10 · SGRO, where:
- **EGRLF** = analyst long-term (3–5yr) predicted earnings growth — the dominant descriptor at 70%
- **EGRO** = trailing earnings growth: normalized 5-year regression slope of annual EPS on time (slope / |mean EPS|)
- **SGRO** = trailing sales growth: normalized 5-year regression slope of annual SPS on time

**Forced deviation.** EGRLF is entirely absent from Sharadar (no analyst estimates). Unlike EYLD where renormalizing was the honest choice, here a proxy is defensible: trailing EGRO and forward EGRLF both target earnings growth at different horizons, and trailing growth is the standard instrument for expected growth when estimates are unavailable. Solution: **proxy EGRLF with EGRO**, giving effective **GRO = 0.90 · EGRO + 0.10 · SGRO**. The caveat: 70% of the intended exposure is trailing, not forward; GRO under-captures expectations-driven growth repricing. See deviations register A2.

**Pre-clip at ±5** (deviations register B5). The normalized growth slope explodes when mean EPS is small but positive — a company earning $0.01/share that grows by $0.05/share has a "normalized slope" of 5, which is not meaningfully different from 100. EGRO and SGRO are clipped to ±5 before compositing (41,815 and 1,673 rows on the reference run). Without this, the 3σ trim bounds are set by artefacts rather than by genuine growth outliers.

**Character.** MoM ρ = 0.9863. Annual inputs are slow-moving by construction; EGRO is a 5-year slope that updates only once per annual filing. ESTU median ≈ −0.081 (plausible — many mature firms have flat or declining EPS in real terms). Quintile spread 1.259.

---

### 9. Momentum (mom)

**Economic role.** Momentum is the empirical finding that recent winner stocks continue to outperform recent losers over medium horizons. The mechanism is contested — behavioral explanations (underreaction, anchoring), risk-based explanations (time-varying premia), and microstructure explanations all have adherents. What is agreed: the signal reverses sharply in the most recent month (short-term reversal), and momentum strategies experience severe crashes during sharp market reversals (2001, 2009, 2020). Momentum is the only USE4 style factor with a deliberate skip period to avoid reversal contamination.

**USE4 specification.** RSTR (Relative Strength): cap-weighted sum of ln(1 + r_it) over the trailing 504 trading days (≈ 2 years), **excluding the most recent 21 trading days** (the skip period), with a 126-day exponential half-life. MOMENTUM = 1.0 · RSTR.

**Construction.** For each stock at each signal date: slice daily excess returns from [t − 504td, t − 21td], apply exponential weights (126d half-life, so returns near the skip boundary get the most weight), and sum the log gross returns. The skip gap [t − 21td, t] is excluded entirely. Min 252 observations. The benchmark is the cap-weighted ESTU excess return from the shared daily artifact, built once in `daily_panel_build.ipynb`.

**Key notes.**
- The 504-day window is the longest of any USE4 descriptor. Combined with the 21-day skip, a stock needs slightly more than 2 years of price history before it receives a momentum score.
- Momentum crashes (2001, 2009, 2020) visibly show up as periods where the Q5–Q1 quintile spread reverses sign. These are not bugs — they are the documented crash structure of momentum factors during sharp market reversals.
- **Lowest stability of all style factors** (MoM ρ = 0.8702; published USE4 range 0.89–0.92). Despite the long window, the half-life weighting keeps momentum responsive enough to produce meaningful rank changes month-to-month. The lab's slightly lower ρ reflects the shorter sample period with more crisis-period weight.

---

### 10. Residual Volatility (resvol)

**Economic role.** Residual Volatility captures idiosyncratic risk — the volatility that remains after stripping out the market-driven component. High-resvol stocks are often small-cap, thinly covered, or undergoing corporate events (M&A rumors, distress, patent disputes). The low-volatility anomaly — that low-residual-volatility stocks outperform on a risk-adjusted basis — is one of the most persistent puzzles in empirical asset pricing and one of the main reasons Residual Volatility is included alongside Beta as a separate factor rather than being subsumed by it.

**USE4 specification.** RESVOL = 0.75 · DASTD + 0.15 · CMRA + 0.10 · HSIGMA, then orthogonalized to Beta:
- **DASTD**: standard deviation of daily excess returns over the trailing 252 days, 42-day exponential half-life
- **CMRA** (Cumulative Range): ln(1 + max_12m_monthly_return) − ln(1 + min_12m_monthly_return) over the trailing 12 months — captures the range of the cumulative return path, not just point-in-time vol
- **HSIGMA**: standard deviation of residuals from the stock's Beta regression, 252-day window, 63-day half-life

**Orthogonalization.** The composite is orthogonalized to `beta_score` via **no-intercept** WLS (weights ∝ √mcap, fit on ESTU, residual applied to all). No intercept because `beta_score` is already CW-mean-zero — an intercept would re-absorb the level that final standardization removes anyway. The weighted inner product ⟨RESVOL, beta_score⟩ is ≈10⁻¹⁷ after orthogonalization. See deviations register B10.

**Upstream dependency.** RESVOL consumes `beta_use4.parquet` for both HSIGMA (reuses the Beta regression residuals) and the orthogonalization. Build Beta first.

**Character.** MoM ρ = 0.9397. Moderately stable — volatility is persistent over medium horizons, but the 42-day half-life on DASTD makes RESVOL responsive to regime changes. ESTU clip rate 0.95% (within the 0.5–2% target band). The low-volatility anomaly signature (negative return spread for high-RESVOL long-short) is visible in quintile diagnostics post-crisis.

---

### 11. Non-linear Beta (nlb)

**Economic role.** NLB captures non-linearities in the market-sensitivity/return relationship. Very-high-beta and very-low-beta stocks have historically earned premia beyond what a linear Beta factor explains — the extreme high-beta end reflects leveraged-equity and speculative-grade effects; the extreme low-beta end may reflect defensive-sector concentration effects. Orthogonalizing to Beta ensures NLB captures purely the curvature, not the level.

**USE4 specification.** NLB = cube of standardized Beta, orthogonalized to Beta. Cubing `beta_score` (which is already mean-zero and unit-variance) amplifies the tails symmetrically — a stock at beta_score = +2 contributes +8 to NLB_raw; a stock at beta_score = −2 contributes −8 — while leaving the center of the distribution nearly unchanged.

**Construction.** Cube `beta_score` → trim to 3σ (CW mean, EW std from ESTU) → orthogonalize to `beta_score` via √mcap-weighted WLS with intercept, fit on ESTU, residual applied to all → restandardize (CW mean 0, EW std 1).

**Upstream dependency.** NLB **must consume the same `beta_use4.parquet`** that enters the risk model, not a locally recomputed beta. Orthogonalizing to a locally recomputed exposure only partially removes the collinearity NLB exists to address. Build Beta first; NLB reads the deliverable via loud loader assertions. See deviations register B11.

**Outlier method.** The generic 3σ trim applies (not percentile winsorization as used for NLS). The cubic distribution has heavier tails than a Gaussian, making a 3σ clip more aggressive: ESTU clip rate is 2.42%, slightly above the 0.5–2% target band (flagged WARN in the audit). If the clip rate drifts past ~3%, switching NLB to 1%/99% percentile winsorization is the natural next step. See deviations register B9.

**Character.** MoM ρ = 0.8723. Low stability by design — NLB captures curvature in the beta distribution, which shifts as stocks move across the β = 0 and β = 2 thresholds. The instability is genuine signal, not noise; the factor's economic role requires responsiveness to changes in extreme market sensitivity.

---

### 12. Non-linear Size (nls)

**Economic role.** NLS captures non-linearities in the size/return relationship that a linear log-market-cap factor misses. The extreme small-cap tail (micro-caps) and the mega-cap tail behave differently from what a linear Size factor predicts — micro-caps face acute illiquidity and information gaps; mega-caps face regulatory scrutiny, index-inclusion mechanics, and concentrated investor bases. NLS isolates this curvature by orthogonalizing to Size.

**USE4 specification.** NLS = cube of standardized Size, orthogonalized to log market cap (the raw `lncap`, not `size_score`). The orthogonalization target is the raw log-mcap rather than the standardized score because the USE4 spec specifies orthogonalization to log_mcap; using size_score produces a slightly different residual.

**Construction.** Cube `size_score` → percentile winsorize at 1%/99% (ESTU quantiles, applied to all) → orthogonalize to `lncap` (raw log-mcap) via √mcap-weighted WLS with intercept, fit on ESTU, residual applied to all → restandardize.

**Outlier method differs from NLB** (deviations register B9). NLS uses **1%/99% percentile winsorization** rather than 3σ trimming. `nls_raw = size_score³` is a heavily skewed cubic: the std is inflated by the very tail being treated, making σ-based bounds inconsistent — a 3σ clip on the skewed distribution leaves the wide tail through while aggressively clipping the narrow tail. Percentile winsorization clips a designed 2.00% exactly and stabilizes the factor without collapsing the quintile structure.

**Upstream dependency.** NLS consumes `size_use4.parquet` — specifically both `lncap` (for the orthogonalization target) and `size_score` (the input to cubing). As with NLB, orthogonalizing to a locally recomputed size exposure only partially removes the collinearity NLS exists to address. See deviations register B11.

**Character.** MoM ρ = 0.9882, nearly as stable as Size itself (0.9967). The cubing amplifies tails but does not change the rank ordering of mid-range stocks, and market cap rank ordering is highly persistent. The slight instability relative to Size comes from the orthogonalization residual introducing small fluctuations around the cubic trend.
