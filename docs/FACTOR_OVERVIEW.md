# Factor Overview — USE4-Style Model

## Model architecture

This pipeline builds a **USE4-style linear factor risk model** for US equities. The model decomposes each stock's return at each month-end signal date into:

```
r_it = f_c,t + Σ_j x_ij,t · f_j,t + e_it
```

where:
- `r_it` is stock i's excess return over the following month
- `f_c,t` is the **Country factor** return (the market intercept)
- `x_ij,t` is stock i's **exposure** to factor j, known at time t
- `f_j,t` is factor j's **return** over the following month (estimated by cross-sectional regression)
- `e_it` is idiosyncratic return

The factors are grouped into three layers:

| Layer | Count | What they capture |
|---|---|---|
| Country | 1 | Common US market return (intercept) |
| Industry | 55 | Industry-relative performance |
| Style | 12 | Cross-sectional return premia |

Every stock gets exactly one country exposure (= 1), exactly one industry exposure (= 1, single membership), and twelve style exposures. The cross-sectional regression that recovers factor returns is WLS with √mcap weights and a cap-weighted zero-sum constraint on the 55 industry factor returns — making industry returns pure relative bets versus the market.

**Two structural facts to internalize before building anything:**

- **One ESTU, everywhere.** All standardization moments (cap-weighted mean, equal-weighted standard deviation) come from the same estimation universe. Factor-specific universes silently break the multi-factor regression that follows.
- **Factor-on-factor inputs are deliberate.** NLB consumes your built Beta deliverable; NLS consumes your built Size deliverable. Orthogonalizing to a locally recomputed exposure only partially removes the collinearity these factors exist to address.

For data-availability gaps relative to USE4 (no analyst estimates, no float, no preferred equity, no GICS, no segment data), see [`data/README.md`](../data/README.md#limitations-relative-to-use4).

---

## Style factor summary

The 12 USE4 style factors, in recommended build order. MoM ρ is the month-over-month rank stability from the reference run (2026-06-11).

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

Suggested path: **size → beta → bp** (one spot fundamental, one time-series, one PIT-join factor — you touch every mechanic), then extract the **daily panel**, then the rest in any order respecting `beta → {resvol, nlb}` and `size → nls`.

---

## In-depth factor discussions

---

### 1. Size

**Economic role.** Size captures the persistent return spread between small-cap and large-cap stocks. Empirically, small caps have outperformed large caps over long horizons in most markets — attributed to liquidity premia, limited analyst coverage, higher idiosyncratic risk, and institutional neglect. In a risk model, Size serves a second, structural purpose: many other factors (Value, Liquidity, Volatility) are strongly correlated with market cap. Controlling for Size ensures those factors isolate their economically distinct signals rather than partly reflecting company scale. Without Size in the model, apparent Liquidity premium partly reflects the small-cap premium; apparent Earnings Yield premia partly reflect the distress concentration in micro-caps. Size is the foundation layer.

**USE4 specification.** `SIZE = 1.0 · LNCAP`, where `LNCAP = ln(total market capitalization)`. One descriptor, weight 1.0 — the simplest factor in the model. The log transformation serves a critical function: raw market cap is right-skewed by several orders of magnitude (Apple at $3T versus a $300M micro-cap differ by 10,000×). Taking the log compresses this to roughly 10 log units of spread and makes the cross-sectional distribution approximately Gaussian, which is necessary for standardization, trimming, and WLS regression to behave as intended.

**Construction.** At each month-end signal date, take the most recent SEP `marketcap` (in millions) via a backward asof join (no look-ahead — only prices known at or before the signal date are used), then compute `lncap = ln(marketcap × 1,000,000)`. Standardize cross-sectionally using cap-weighted mean and equal-weighted standard deviation from the ESTU. Both `lncap` and `size_score` are saved to the deliverable: `lncap` is needed downstream as a WLS regression weight (`exp(lncap/2) = √mcap`) and as the orthogonalization target for NLS; `size_score` is the exposure that enters the risk model.

**Standardization convention.** The cap-weighted mean of `size_score` equals zero by construction — the cap-weighted portfolio has a zero Size score, or equivalently the "average stock by market weight" has neutral Size. The equal-weighted standard deviation is 1.0. This convention is shared by all 12 style factors and is required for the cross-sectional regression to have a meaningful country factor (intercept).

**Key deviations.** Two choices differ from the generic USE4 trim recipe, both documented in deviations register B8. First, the trim center uses the **equal-weighted mean** rather than the cap-weighted mean: for `lncap`, the cap-weighted mean sits roughly 4 log units above the equal-weighted mean (large-caps dominate cap-weight), so a CW-centered 3σ band clips approximately 20% of small-cap ESTU rows — the wrong tail entirely. Second, the multiplier is **4σ rather than 3σ**: `lncap` is approximately Gaussian (log of a log-normal), so a 4σ bound clips less than 0.1% of ESTU in practice while preserving the full legitimate large-cap tail. Separately, the factor uses total market cap rather than float-adjusted market cap because Sharadar provides only total cap (deviations register A3), a limitation that pervades ESTU weights and the Size exposure alike.

**Character.** Highest MoM stability of any factor (ρ = 0.9967) — market cap changes gradually except during extreme events like crashes or mega-mergers. The median ESTU stock has a negative `size_score` because most stocks by count are small; the cap-weighted mean, which equals zero by construction, sits firmly in large-cap territory. ESTU clip rate 0.05% (404 of 822,918 rows on the reference run), well within any reasonable bound.

---

### 2. Beta

**Economic role.** Beta measures a stock's sensitivity to market-wide moves — the fraction of its return variance explained by the market factor. In USE4 it is the highest-signal style factor by average |t|-statistic (3.96–4.02 per the Empirical Notes) and the only style factor with non-trivial cross-sectional correlation to the cap-weighted ESTU (0.77–0.90). The reason is structural: tilting a portfolio toward high-beta stocks is mechanically similar to levering the market. Beta and the Country factor are deliberately distinct: the Country factor describes the common US market return; Beta describes each stock's *sensitivity* to that return. A high-beta tilt on a long-only portfolio carries more market risk than a pure market exposure of the same cost — that additional systematic risk is what Beta isolates and prices.

**USE4 specification.** The beta descriptor is the OLS slope in the regression `r_it − r_ft = α + β·R_t + e_t`, estimated over 252 trailing trading days (approximately one calendar year) with a 63-day exponential half-life. `R_t` is the cap-weighted excess return of the ESTU — the estimation universe's own return, not an external index. Final factor: `BETA = 1.0 · BETA` (single descriptor).

**Construction.** The exponential half-life weights recent observations more heavily: a return 63 days ago gets exactly half the weight of today's return. For each stock at each signal date, slice the prior 252 trading days of daily excess returns (stock and benchmark both in excess of the Ken French risk-free rate), attach exponential weights, and solve the weighted normal equations. Minimum 63 observations; stocks with fewer get a null beta and are excluded from the risk model for that date. The benchmark uses **lagged market caps** (prior-day `mcap`) to compute weights, avoiding the contemporaneous-weight bias that would arise if today's cap weight reflected today's price move before the return is computed. Standardize cross-sectionally (CW mean 0, EW std 1 from ESTU).

**Key facts.** The cap-weighted mean of raw beta across ESTU stocks is identically 1.0 — the ESTU portfolio's beta against itself must equal one by arithmetic. This means the CW mean of `beta_score` after standardization is zero (standard convention) but the raw betas are centered at 1 rather than 0. Build Beta before NLB and Residual Volatility; both consume `beta_use4.parquet` as an upstream input. Beta is not explicitly winsorized in the USE4 spec (unlike NLB/NLS); the general 3σ trim from Methodology Notes §2.2 is applied as a lightweight guard.

**Character.** MoM ρ = 0.9458, moderate stability relative to the fundamental factors. The 252-day window with exponential decay makes month-to-month changes gradual because the oldest observations roll off each month while receiving lower weight by design, but the exponential half-life keeps Beta more responsive to recent regime shifts (volatility clustering, correlation spikes) than a naive rolling OLS would be. Published USE4 range: 0.96–0.97; the lab's slightly lower ρ reflects the shorter calibration sample with more crisis-period weight.

---

### 3. Book-to-Price (bp)

**Economic role.** Book-to-Price (the inverse of Price/Book) is the canonical value factor: stocks trading cheaply relative to their accounting book value have historically outperformed growth stocks over long horizons. The mechanism is actively debated — competing explanations include rational mispricing that reverts, compensation for financial distress risk, and mean-reverting investor sentiment. What is agreed is the empirical spread across markets and decades. In USE4, BP is one of three value-oriented factors alongside Earnings Yield and Dividend Yield; together they form the "value cluster." BP is the purest balance-sheet value signal, anchoring to the stock of net assets rather than the flow of earnings or income. Its complement factors (EYLD, DYLD) capture cash-generation and income signals from the income statement.

**USE4 specification.** `BTOP = book equity / current market cap`. Book equity is defined as total shareholders' equity attributable to the parent minus preferred equity. Market cap is contemporaneous at the signal date, not the filing date — which is critical: a filing from six months ago paired with a six-months-stale price would produce an incorrect ratio that combines stale fundamentals with stale price action in a misleading way.

**Construction.** Load SF1 with `dimension = "ARQ"` (as-reported quarterly — each row reflects what was actually published, not restated figures), point-in-time keyed on `datekey` with an 18-month staleness window. The PIT key is `datekey` (the filing release date), never `calendardate` (the period end): using `calendardate` would introduce 30–90 days of look-ahead bias per filing because quarterly results are published weeks after the period closes. Book equity = SF1 `equity` field. Market cap = `estu_monthly.mcap` at signal date — always the signal-date price, never SF1's filing-date market cap. Standardize cross-sectionally.

**Forced deviations.** Preferred equity is absent from Sharadar SF1 (deviations register A4). USE4 defines book equity as total equity minus preferred equity; the `equity` field in SF1 is already the common equity figure per GAAP (common stock + additional paid-in capital + retained earnings + accumulated other comprehensive income), so it is conceptually correct for most companies. The error is concentrated in financials and utilities where preferred stock is most common, and damps the factor (slightly overstating BE, understating BTOP) rather than inverting the ordering. Negative book equity is retained rather than nulled (deviations register B3): approximately 49,043 ARQ rows carry negative equity, arising from buy-back-heavy firms and distressed leveraged situations. Treating them as missing would silently delete a coherent economic cohort; retaining them as negative BTOP correctly places them below low-BTOP growth names in the value ordering.

**Character.** MoM ρ = 0.9773. Book value is a balance-sheet stock that changes slowly — retained earnings accumulate, depreciation is predictable, most quarters involve no large structural change. The main source of month-to-month variation is the price denominator, which updates daily but is sampled only at signal-date values. Median BTOP ≈ 0.41 with strictly monotone quintiles across reference dates.

---

### 4. Earnings Yield (eyld)

**Economic role.** Earnings Yield (E/P) is the USE4 Empirical Notes' "most significant factor in explaining value-related return differences in U.S. equities." High E/P stocks trade at low earnings multiples — classically interpreted as undervaluation, or as compensation for higher cyclicality and distress risk. The economic intuition is straightforward: investors are implicitly paying a smaller multiple of current earnings for the forward cash flow stream, either because the market discounts future growth rates or because there is systematic mispricing that reverts. The USE4 composite is deliberately blended between forward-looking analyst consensus (which dominates at 75%) and trailing signals because markets are forward-looking: the multiple at which investors capitalize current earnings depends critically on their expectation of future earnings. This blend is what makes EYLD more predictive than a simple trailing E/P.

**USE4 specification.** `EYLD = 0.75 · EPFWD + 0.15 · CETOP + 0.10 · ETOP`. EPFWD is 12-month forward EPS over market cap (analyst consensus). CETOP is trailing 12-month cash earnings (net income + depreciation) over price. ETOP is trailing 12-month net earnings over market cap. The 75% forward weight reflects USE4's empirical finding that the forward component dominates in cross-sectional predictive power.

**Forced deviation (material).** EPFWD is entirely absent from Sharadar — no analyst consensus data exists in this data stack. Proxying EPFWD with trailing EPS would duplicate ETOP with no added information while falsely implying forward content. The honest and only defensible choice is to drop EPFWD and renormalize over the remaining descriptors: **EYLD = 0.60 · CETOP + 0.40 · ETOP**, preserving the 0.15:0.10 ratio between CETOP and ETOP. The result is a pure trailing value measure. What is lost is the forward-looking revision signal — the part of the factor that would capture analyst estimate revisions, earnings surprise momentum, and expected versus realized growth divergences. See deviations register A1.

**Construction.** TTM figures are constructed as 4-quarter rolling sums over `calendardate` from the ARQ dimension (the SF1 extract carries only ARQ, not the TTM convenience dimension — deviations register A8). Cash earnings = `netinc + depamor` (the standard first-order cash-earnings approximation, adding back the non-cash depreciation and amortization charge to reported net income — deviations register A5). CETOP and ETOP denominators use `estu_monthly.mcap` at signal date — never SF1's filing-date market cap, which could be months stale and would combine old earnings with old prices rather than the current valuation level. Four complete quarters are required for EYLD (no partial-year annualization). Negative earnings are kept; the 3σ trim handles extremes. Standardize cross-sectionally.

**Character.** MoM ρ = 0.9630. Trailing earnings are sticky, but large write-downs, impairment charges, and cyclical swings (particularly energy and financials) can produce jumps. ESTU median raw eyld ≈ 0.059 (a realistic US earnings yield in the 2020s), with strictly monotone quintiles and quintile spread 0.260.

---

### 5. Dividend Yield (dyld)

**Economic role.** Dividend Yield reflects the income component of total return and a signaling effect about managerial confidence: companies that pay and grow dividends tend to be profitable, mature, and confident enough about future cash generation to commit to a recurring payment that is costly to cut. In USE4, Dividend Yield is a value-adjacent factor that differentiates mature, cash-generative businesses from high-growth names. It is distinct from Earnings Yield in an important way: a company can have high EYLD (low P/E) while paying no dividend because it reinvests all earnings, and another company can have high DYLD but moderate EYLD if it pays out most of a high income stream. The two factors each tell a different part of the cash distribution story.

The distinctive characteristic of DYLD — versus every other style factor — is that the majority of the universe (approximately 63%) pays no dividend, creating a large mass point at zero in the raw distribution. How the factor handles non-payers is the single most consequential construction decision.

**USE4 specification.** `DTOP = trailing 12-month DPS / current price`. Single descriptor.

**Construction.** TTM DPS is the 4-quarter sum of split-adjusted DPS from ARQ, requiring at least 1 complete quarter (so zero-payers survive with `dyld = 0.0`, and the factor retains full coverage). DPS is rebaselined across split-affected quarters using cumulative split factors from the ACTIONS table: 217,223 split-affected quarters on the reference run. A stock that split 2-for-1 mid-year has its pre-split DPS halved to be comparable to its post-split price basis. The denominator is the signal-date price from SEP.

**Zero-payers are zeros, not nulls (deviations register B4).** This is the central design decision for DYLD. A company that pays no dividend has `dyld = 0.0` — a real economic statement that this company currently distributes nothing to shareholders. Treating zero-payers as missing would shrink coverage by approximately 63% and transform DYLD from a factor about the distribution policy of all investable companies into a factor about the yield level among companies that have already chosen to pay. The quintile structure reflects this design: quintiles Q1 and Q2 have mean exactly 0.0 (the zero-payer mass); Q3 through Q5 strictly increase to a median of approximately 0.056 on the reference run.

**Corrupt-DPS guard (deviations register B2).** Raw SF1 `dps` contains split-era artefacts implying physically impossible yields. Any quarterly DPS implying a single-quarter yield exceeding 50% is nulled at source (1,292 filings on the reference run); the same 50% threshold backstops the TTM yield (8,700 rows nulled). No legitimate dividend policy produces a 50%-per-quarter yield — these are share-count mismatches in the vendor data, not real dividends. Nulling (rather than capping) keeps fabricated values out of the TTM sum entirely.

**Character.** Highest MoM stability of all fundamental factors (ρ = 0.9927). Dividend policy is extremely sticky — cuts and initiations are infrequent and deliberate. The main source of month-to-month variability is the price denominator, which updates daily. Max post-trim dyld ≈ 0.202 on the reference run; median ≈ 0.00285 across all ESTU stocks including zero-payers.

---

### 6. Leverage (lev)

**Economic role.** Leverage captures the financial risk embedded in a company's capital structure. Highly levered firms have more volatile equity returns because debt service is a fixed cost that magnifies equity volatility — a firm with a 50% debt/equity ratio sees a 1% change in firm value translate to a 2% change in equity value. Beyond amplification, high leverage raises distress risk: a string of bad outcomes that would merely impair an unlevered firm can bankrupt a levered one. In a risk model, Leverage loads on the spread between levered and unlevered equity returns independently of Beta and Volatility, because Beta reflects the systematic component of the equity's return amplification while Leverage also captures the idiosyncratic structural risk of the balance sheet itself.

**USE4 specification.** `LEV = 0.75 · MLEV + 0.15 · DTOA + 0.10 · BLEV`, where:
- **MLEV** (market leverage): `(market equity + preferred equity + long-term debt) / market equity`. Uses market equity in the denominator — the going-concern value of equity — making this measure forward-looking and responsive to current market conditions.
- **DTOA** (debt to assets): `total debt / total assets`. A pure accounting ratio, bounded in [0, 1], capturing the asset-coverage perspective on leverage.
- **BLEV** (book leverage): `(book equity + preferred equity + long-term debt) / book equity`. Uses book values throughout, providing a historical-cost anchor that changes slowly.

MLEV gets the dominant 75% weight because market equity is forward-looking: it reflects the current burden of debt relative to going-concern value rather than the historical cost basis that book measures use. The three-descriptor blend is deliberately multi-lens: MLEV can spike during a stock price collapse even if no new debt is issued; DTOA provides stability; BLEV captures the historical structural choice of debt financing.

**Construction.** The sub-descriptors live on incompatible scales: MLEV and BLEV both floor at 1.0 (minimum for an unlevered firm with no debt) and can reach 20+ for highly levered firms; DTOA is bounded strictly in [0, 1]. A naive weighted sum would give MLEV far more than its intended 75% share purely from scale differences, not from economic meaning. The solution is to **standardize each sub-descriptor cross-sectionally before compositing** (deviations register B7): compute CW-mean-zero, EW-std-1 scores for each sub-descriptor individually within ESTU, then apply the 0.75/0.15/0.10 weights. Post-standardization the CW mean of each sub-descriptor is machine-zero (residual ≤ 6×10⁻¹⁶). Source: SF1 ARQ, PIT-keyed on `datekey`, 18-month staleness window.

**Null debt treated as zero (deviations register B6).** `debtnc` is null in 20.3% of ARQ rows. For balance-sheet debt, absence of a line item means the firm carries none — a company that reports total assets and equity without reporting long-term debt is telling you it has none. Treating nulls as missing would delete one-fifth of the universe, biased precisely toward the unlevered firms that Leverage must rank lowest. Min MLEV and BLEV after applying this rule = 1.0000 exactly (zero-debt companies), confirming correctness.

**Preferred equity absent (deviations register A4).** With PE = 0: MLEV numerator loses the preferred equity term (understating the claim on firm value senior to common equity); BLEV's algebra simplifies because `BE + PE = EQUITY` becomes `EQUITY + DEBTNC / EQUITY`. The error damps the factor for financials and utilities — the heaviest preferred-stock issuers still sort to Q5 because their debt load alone drives the composite.

**Character.** MoM ρ = 0.9908. Capital structure changes slowly; leverage is highly persistent. The ESTU clip rate is 2.71%, above the 0.5–2% target band (flagged WARN in the audit) — this reflects Leverage's structurally heavy right tail (distressed firms, financial companies) rather than a parameter error. Quintile spread 1.755, strictly monotone on reference dates.

---

### 7. Liquidity (liq)

**Economic role.** Liquidity captures the cost of trading a stock. Illiquid stocks command a return premium — investors demand compensation for the trading friction they incur when building or exiting a position. But liquidity is also a noise amplifier: returns of thinly traded names reflect microstructure effects (bid-ask bounce, price impact of small trades) as much as fundamental information. In a risk model, Liquidity isolates return variation driven by trading friction, independently of Size (with which it is correlated — small stocks tend to be illiquid — but not identical; many mid-cap stocks are highly liquid while some large-cap stocks in thinly traded sectors are not). Controlling for both Size and Liquidity separately ensures each factor captures its intended source of return variation.

**USE4 specification.** `LIQ = 0.35 · STOM + 0.35 · STOQ + 0.30 · STOA`, where each descriptor is the **natural log of cumulative share turnover** (volume / shares outstanding) over a different horizon:
- **STOM**: log turnover over the trailing month (~21 trading days)
- **STOQ**: log turnover over the trailing quarter (~63 trading days)
- **STOA**: log turnover over the trailing year (~252 trading days)

Taking the **log of the sum** (not the mean) serves two purposes: it compresses the enormous range between a micro-cap with 0.01% daily turnover and a mega-cap during a short squeeze; and it makes the cross-sectional distribution approximately Gaussian, which is required for well-behaved standardization and factor exposures. The three-horizon blend is deliberate: STOM captures current trading activity and short-term demand; STOA captures structural liquidity — the trading environment a stock operates in over the long run. A stock that traded heavily last month due to an M&A rumor but is normally illiquid won't score as a liquid stock because STOQ and STOA carry 65% of the weight.

**Construction.** Share turnover = cumulative volume over the window / shares outstanding on the signal date. Shares outstanding is derived from `marketcap / closeunadj` per SEP row (Sharadar does not directly provide shares outstanding in SEP; it is imputed from unadjusted price and market cap). The three-horizon sums are computed over the shared daily panel and joined at signal dates. Standardize cross-sectionally.

**Key data issue (deviations register A7).** 20,710 SEP rows (0.057%, 2,221 tickers) show turnover > 1 — physically impossible under normal market conditions, since a stock cannot have more volume than shares outstanding in a normal trading day. These are vendor share-count artefacts. The log transform plus 3σ trim absorb them without row-level surgery; at 0.057% of rows they are too sparse to move monthly medians. The median daily turnover ≈ 0.00461 sits inside the 0.001–0.010 plausibility band.

**Character.** MoM ρ = 0.9706. A stock's trading environment doesn't change dramatically month-to-month. Short-term events (earnings announcements, index additions) can spike STOM, but STOQ and STOA dampen them. ESTU clip rate 0.95%, within the target band. The all-universe in-band fraction (84.3%) is materially lower than the ESTU in-band fraction (98.9%) because non-ESTU micro-caps legitimately have extreme negative Liquidity scores — they are illiquid by construction.

---

### 8. Growth (gro)

**Economic role.** Growth captures the market premium for companies with strong earnings and sales expansion trajectories. High-growth firms typically command high price multiples because investors are willing to pay now for future earnings; the factor isolates whether that growth premium is warranted by the actual fundamental trajectory. In USE4, Growth is negatively correlated with Earnings Yield and Book-to-Price — high-growth firms tend to have high P/E and P/B, so Growth exposure is the complement of the value cluster. The factor is also the most affected by the missing analyst data, because 70% of its intended exposure depends on forward-looking analyst consensus growth estimates that are unavailable in public data.

**USE4 specification.** `GRO = 0.70 · EGRLF + 0.20 · EGRO + 0.10 · SGRO`, where:
- **EGRLF**: analyst long-term (3–5 year) predicted earnings growth — the dominant descriptor at 70%
- **EGRO**: trailing earnings growth — normalized 5-year regression slope of annual EPS on time, divided by |mean EPS|
- **SGRO**: trailing sales growth — normalized 5-year regression slope of annual sales per share on time, divided by |mean SPS|

The normalization (dividing by |mean| of the series) converts the absolute slope into a percentage-of-magnitude growth rate, making the signal comparable across companies of vastly different earnings levels.

**Forced deviation (material).** EGRLF is entirely absent from Sharadar. Unlike EYLD where renormalizing was the honest choice, here a proxy is defensible: trailing EGRO and forward EGRLF both target the same quantity — earnings growth — at different horizons, and trailing growth is the standard instrument for expected growth when estimates are unavailable. Solution: **proxy EGRLF with EGRO**, giving effective **GRO = 0.90 · EGRO + 0.10 · SGRO**. The caveat stands: 70% of the intended exposure becomes trailing rather than forward, so GRO under-captures the expectations-driven growth repricing that gives the USE4 composite its forward-looking character. See deviations register A2.

**Pre-clip at ±5 (deviations register B5).** The normalized growth slope explodes when mean EPS or mean SPS is small but positive — a company earning $0.01/share that grows by $0.05/share has a normalized slope of 5, which is not meaningfully different from 100 and produces no useful signal. EGRO and SGRO are clipped to ±5 before compositing (41,815 EGRO rows and 1,673 SGRO rows on the reference run). Without this pre-clip, the 3σ trim bounds are determined by artefacts rather than genuine growth outliers.

**Character.** MoM ρ = 0.9863. Annual inputs are slow-moving by construction: EGRO is a 5-year OLS slope that updates only once per annual filing. ESTU median ≈ −0.081 (plausible — many mature firms have flat or declining real EPS). Quintile spread 1.259, strictly monotone.

---

### 9. Momentum (mom)

**Economic role.** Momentum is the empirical finding that recent winner stocks continue to outperform recent losers over medium horizons (3–12 months). The mechanism is contested: behavioral explanations emphasize underreaction (investors update too slowly to new information, so prices continue drifting toward fair value for months after an earnings surprise); risk-based explanations argue for time-varying risk premia; microstructure explanations point to serial correlation in institutional order flow. What is agreed: the signal is robust across markets and eras, but reverses sharply in the most recent month (short-term reversal, driven by liquidity effects) and experiences severe crashes during sharp market reversals (2001, 2009, 2020 — when recent winners get sold to meet redemptions and past losers snap back). The skip period and exponential weighting in USE4's design address both known pathologies.

**USE4 specification.** `RSTR` (Relative Strength): cap-weighted sum of `ln(1 + r_it)` over the trailing 504 trading days (approximately 2 years), **excluding the most recent 21 trading days** (the skip period), with a 126-day exponential half-life. `MOMENTUM = 1.0 · RSTR`.

The design choices each have an explicit rationale:
- **504 days (~2 years)** captures the full medium-term momentum horizon beyond which return correlation reverses to long-term mean reversion.
- **Skip 21 days** excludes the most recent month's return, which reverses due to liquidity and bid-ask effects rather than reflecting genuine momentum.
- **126-day half-life** weights recent returns within the window more heavily, consistent with the empirical finding that shorter-horizon returns are more predictive of near-term continuation than 18-month-old returns.

**Construction.** For each stock at each signal date: slice daily excess returns from [t − 504 trading days, t − 21 trading days], apply exponential weights (126-day half-life, so returns near the skip boundary receive the most weight), and sum the log gross returns. The skip gap [t − 21 trading days, t] is excluded entirely — not down-weighted, excluded. Minimum 252 observations. The benchmark is the cap-weighted ESTU excess return from the shared daily artifact.

**Key notes.** The 504-day window is the longest of any USE4 descriptor. Combined with the 21-day skip and the 252-observation minimum, a stock needs slightly more than 2 years of price history before it receives a momentum score — appropriate because the momentum strategy's theoretical underpinning (underreaction to multi-quarter earnings news cycles) only operates on names with sufficient market history. Momentum crashes (2001, 2009, 2020) visibly appear in quintile diagnostics as periods where the Q5–Q1 spread reverses sign; these are not bugs but the documented crash signature.

**Character.** Lowest MoM stability of all style factors (ρ = 0.8702; published USE4 range: 0.89–0.92). Despite the long window, the 126-day half-life keeps Momentum responsive enough to produce meaningful rank changes month-to-month. The lab's slightly lower ρ reflects the shorter calibration sample with disproportionate crisis-period weight.

---

### 10. Residual Volatility (resvol)

**Economic role.** Residual Volatility captures idiosyncratic risk — the volatility that remains after stripping out the market-driven (systematic) component of return. High-resvol stocks are often small-cap, thinly covered by analysts, or undergoing corporate events (M&A speculation, patent disputes, distress), producing large unpredictable swings not explained by the market's moves. The low-volatility anomaly — that low-residual-volatility stocks have historically outperformed on a risk-adjusted basis, especially after controlling for Size — is one of the most persistent puzzles in empirical asset pricing. One explanation: institutional investors constrained to benchmark-relative tracking error systematically overpay for high-idiosyncratic-volatility stocks (lottery-ticket demand), pushing their prices above fair value. Residual Volatility is included alongside Beta as a separate factor because the two measure distinct things: Beta captures co-movement with the market; Residual Volatility captures the magnitude of the remaining unexplained swings.

**USE4 specification.** `RESVOL = 0.75 · DASTD + 0.15 · CMRA + 0.10 · HSIGMA`, then orthogonalized to Beta:
- **DASTD** (Daily Annualized Std): standard deviation of daily excess returns over the trailing 252 days, with a 42-day exponential half-life. This is the workhorse descriptor — realized volatility over a 1-year window with recent emphasis.
- **CMRA** (Cumulative Range): `ln(1 + max_monthly_cum_return) − ln(1 + min_monthly_cum_return)` over the trailing 12 months. This measures the *range* of the cumulative return path: a stock that spent half the year up 30% and the other half down 30% has a large CMRA regardless of current price. CMRA captures the width of the stock's experienced outcome space.
- **HSIGMA** (Historical Sigma): standard deviation of residuals from the stock's Beta regression, 252-day window, 63-day half-life. Explicitly idiosyncratic volatility — the part of daily return that the Beta regression cannot explain.

**Orthogonalization.** The composite is orthogonalized to `beta_score` via **no-intercept** WLS (weights ∝ √mcap, fit on ESTU, residual applied to all stocks). No intercept is used because `beta_score` is already CW-mean-zero — an intercept term in the orthogonalization regression would re-absorb the level that final standardization removes anyway. The defining property holds to machine precision post-orthogonalization: the weighted inner product ⟨RESVOL, beta_score⟩ ≈ 10⁻¹⁷ (see deviations register B10).

**Upstream dependency.** RESVOL consumes `beta_use4.parquet` for both HSIGMA (the Beta regression residuals can be reused) and the orthogonalization step. Build Beta first.

**Character.** MoM ρ = 0.9397. Moderately stable — volatility is persistent over medium horizons, but the 42-day half-life on DASTD makes RESVOL responsive to regime changes. ESTU clip rate 0.95%, within the target band. The low-volatility anomaly signature (negative return spread for high-RESVOL long-short) is visible in quintile diagnostics in post-crisis periods.

---

### 11. Non-linear Beta (nlb)

**Economic role.** NLB captures non-linearities in the market-sensitivity/return relationship that a linear Beta factor cannot model. The return premia at the extreme ends of the beta distribution — very-high-beta and very-low-beta stocks — differ from what a simple linear extrapolation from mid-range beta stocks would predict. Very-high-beta stocks (β > 2) often reflect leveraged equity structures or speculative names whose effective market sensitivity is non-stationary; very-low-beta stocks (β < 0, defensive sectors) may face concentrated regulatory, rate, or sector-rotation effects not captured by the linear slope. Including NLB in the model improves the regression's fit at the tails and absorbs a return component that would otherwise leak into the residual. Orthogonalizing to Beta ensures NLB captures purely the curvature, not the level.

**USE4 specification.** `NLB = cube of standardized Beta, orthogonalized to Beta`. Cubing `beta_score` (which is already mean-zero and unit-variance) amplifies the tails symmetrically and non-linearly: a stock at `beta_score = +2` contributes +8 to NLB_raw; a stock at `beta_score = −2` contributes −8. The cubic is symmetric, so it amplifies both high-beta and low-beta tails without distorting mid-range stocks (near `beta_score = 0`, the cube ≈ 0). This is the natural mathematical function for capturing symmetric curvature around the mean.

**Construction.** Cube `beta_score` → trim to 3σ (CW mean, EW std from ESTU) → orthogonalize to `beta_score` via √mcap-weighted WLS **with intercept** (unlike RESVOL, here an intercept is appropriate because NLB_raw is not centered at zero before orthogonalization), fit on ESTU, residual applied to all → restandardize (CW mean 0, EW std 1).

**Upstream dependency.** NLB **must consume the same `beta_use4.parquet`** that enters the risk model, not a locally recomputed beta score. If NLB orthogonalizes against a locally recomputed beta (same regression but slightly different trim or standardization inputs), the residual does not precisely zero out the correct factor exposure, and the collinearity NLB exists to remove is only partially addressed. Build Beta first; NLB reads the deliverable. See deviations register B11.

**Outlier method.** The generic 3σ trim applies (deviations register B9). The cubic distribution has heavier tails than a Gaussian, making a 3σ clip more aggressive: ESTU clip rate is 2.42%, slightly above the 0.5–2% target band (flagged WARN). If the clip rate drifts past approximately 3%, switching NLB to 1%/99% percentile winsorization is the natural next step.

**Character.** MoM ρ = 0.8723. Low stability by design — NLB captures curvature in the beta distribution, which shifts as stocks move across the β = 0 and β = 2 thresholds. A stock that transitions from slightly-above-market beta to very-high beta undergoes a large NLB change because the cubic amplifies the transition. The instability is genuine signal, not noise.

---

### 12. Non-linear Size (nls)

**Economic role.** NLS captures non-linearities in the size/return relationship that a linear log-market-cap factor misses. The return behavior of extreme micro-caps and extreme mega-caps diverges from what a linear Size exposure would predict. Micro-caps face acute illiquidity, information gaps, analyst neglect, and institutional exclusion — creating a liquidity/neglect premium on top of (but different from) the linear size premium. Mega-caps face regulatory scrutiny, antitrust risk, index-inclusion mechanics, and concentrated institutional bases whose flows create price effects not driven by fundamentals. NLS isolates this curvature by orthogonalizing to the linear Size exposure, ensuring it captures only the extra-linear tail behavior.

**USE4 specification.** `NLS = cube of standardized Size, orthogonalized to log market cap (lncap)`. The orthogonalization target is the **raw log-mcap** (`lncap`), not `size_score`. This is explicit in the USE4 specification: orthogonalizing to `lncap` rather than `size_score` produces a subtly different residual because the standardization of `size_score` introduces a scale change, and the USE4 spec uses the pre-standardization log-cap as the collinearity target.

**Construction.** Cube `size_score` → **percentile winsorize** at 1%/99% (ESTU quantiles, applied to all) → orthogonalize to `lncap` (raw log-mcap) via √mcap-weighted WLS with intercept, fit on ESTU, residual applied to all → restandardize.

**Outlier method differs from NLB (deviations register B9).** NLS uses **1%/99% percentile winsorization** rather than 3σ trimming. `nls_raw = size_score³` is a heavily right-skewed cubic: the standard deviation is inflated by the very tail being treated, making σ-based bounds inconsistent. A 3σ clip on this skewed distribution would leave the wide tail through while aggressively clipping the narrow tail — the opposite of the intended behavior. Percentile winsorization clips a designed 2.00% exactly and stabilizes the factor without collapsing the quintile structure.

**Upstream dependency.** NLS consumes `size_use4.parquet` — specifically both `lncap` (for the orthogonalization target) and `size_score` (the input to cubing). As with NLB, orthogonalizing against locally recomputed Size values only partially removes the collinearity NLS exists to address. See deviations register B11.

**Character.** MoM ρ = 0.9882, nearly as stable as Size itself (0.9967). The cubing amplifies the tails but does not alter the rank ordering of mid-range stocks, and market cap rank ordering is highly persistent. The slight instability relative to Size comes from the orthogonalization residual introducing small fluctuations around the cubic trend.

---

## Industry factors (consolidated)

### Purpose and structure

The 55 industry factors absorb common return variation correlated across stocks in the same industry but uncorrelated with the market or with systematic style tilts. Without industry factors, a model that happened to overweight Technology stocks would conflate the systematic Tech-sector move with its style exposures — the cross-sectional regression cannot cleanly identify, say, an Earnings Yield premium if high-E/P firms happen to cluster in one industry that month.

**Exposures are dummies, not descriptors.** Every stock gets exactly one industry exposure of 1.0 (single membership); all others are 0. There is no standardization step, no trim, no MoM ρ. The validation battery is membership-shaped rather than distribution-shaped: coverage, floor, concentration, static-map integrity, and spot checks.

**Identification constraint.** Every stock also has Country exposure = 1.0. If you stack a constant column and 55 binary industry columns, the 55 industry columns sum exactly to the constant column — the design matrix is rank-deficient by one. USE4 resolves this with the **cap-weighted zero-sum constraint on industry factor returns**:

```
Σ_j  w_jt · f_ind,j,t  =  0        w_jt = cap weight of industry j in ESTU at t
```

This constraint identifies the 55 industry factor returns relative to the market (the Country factor absorbs the market-level return; industry factors describe only industry-relative performance). Implementation: exact reparametrization — eliminate the largest-cap-weight industry at each date (most numerically stable choice), express its factor return as the cap-weighted linear combination of the other 54, and solve the reduced WLS system. See the Country factor section below.

### Why 55 factors, not 60

USE4's 60-factor GICS-based scheme was engineered for the 2011 market. Sector tectonics since then are non-trivial: software and internet names are an order of magnitude larger by market cap; coal, paper, and department stores have hollowed out. A modern rebuild **should not reproduce the 2011 60-list verbatim even where it could** — the published selection criteria (economic intuition, statistical significance, explanatory power, minimum size) matter more than the specific labels.

The factors were engineered from Sharadar's `industry` field (151 observed atoms in the full ticker universe, Morningstar-style taxonomy, 100% ESTU coverage) aggregated by the hand-engineered `industry_scheme.csv` into 55 factors. The alternative fields (`famaindustry`, `sicsector`) were evaluated and rejected: FF48's "Business Services" holds 328 members (far too coarse); SIC's technology granularity is 1980s-vintage and misses modern software distinctions entirely.

### Minimum-size floor

USE4 states that thin industries are avoided because they produce unstable estimates, but gives no specific floor values. The floor chosen: **median ≥ 15 and minimum ≥ 8 ESTU members over the full 1999–2026 sample**. With approximately 2,500 ESTU members and 55 factors, this averages roughly 45 members per factor. Four declared exceptions are kept despite the floor:

| Exception | Justification |
|---|---|
| Internet & Catalog Retail | ~6 median members, but ~$3T of cap (Amazon dominates); cap weight is the relevant size for a cap-weighted regression |
| Managed Health Care | Explicitly separated in the USE4 2011 list; economically distinct from other health insurance lines |
| Airlines | Cyclically important, economically coherent sector; small member count reflects consolidation, not classification noise |
| Industrial Conglomerates | Single-membership forces multi-industry names here; the bucket is conceptually coherent even if small |

### Merges relative to USE4's 60-list

Several USE4 2011 factors were merged because the underlying Sharadar atoms do not support a clean split, or because the combined member count falls below the floor:

- **Drilling → Equipment & Services**: insufficient Sharadar granularity to cleanly separate Oil Drilling from broader Oilfield Services
- **Paper + Forest → Construction Materials**: both sectors hollowed out post-2011; combined member counts remain viable
- **Both metals factors → Metals & Mining**: Gold/Silver/Diversified Metals collapsed into one factor; member counts support only one metals factor at the floor
- **Durables/Apparel/Leisure consolidated**: three thin USE4 factors combined into one consumer goods manufacturing factor
- **Retail consolidated**: specialty retail sub-types combined; single membership limitation makes fine retail distinctions spurious
- **Telecom 2→1**: no wireless/wireline split available in Sharadar atoms; one Telecommunications factor

### Splits relative to USE4's 60-list

Two USE4 2011 factors were split because they became enormous relative to the rest of the scheme — a 150-member factor has the same degrees of freedom in the WLS as a 15-member factor, but economically these large factors conflated distinct business models:

- **Diversified Financials → Capital Markets & Asset Management / Consumer Finance & DFS**: asset managers and broker-dealers versus consumer lenders and digital financial services — distinct return dynamics
- **Software → Application Software / Infrastructure & Gaming**: enterprise application software versus systems/infrastructure and gaming — distinct revenue model, customer base, and macro sensitivities

These splits address 2026 realities USE4 could not anticipate in 2011, when these categories were smaller. The criteria applied are USE4's own: economic intuition, statistical significance, explanatory power, and minimum size.

### Classification limitations

**GICS absent (deviations register A9).** GICS (Global Industry Classification Standard) is licensed and unavailable from Sharadar. The 55-factor scheme approximates USE4's 60-factor GICS-based list using Sharadar's Morningstar-style `industry` atoms. The fail-loudly guard (deviations register C7) ensures any new atom introduced by a vendor taxonomy update causes an explicit build failure rather than a silent default bucket.

**No segment data — single 0/1 membership (deviations register A10).** USE4 reports that approximately 63% of the estimation universe by cap weight had multiple (fractional) industry exposures, derived from business-segment reporting via cross-sectional regressions on segment assets and sales. Sharadar carries no business-segment data; the fractional machinery is unbuildable. Every stock receives a single 0/1 membership. The bias concentrates in conglomerates, diversified financials, and integrated energy. Hand-curating fractional exposures for known conglomerates was considered and rejected: a subjective, unreproducible input is worse than a documented uniform rule.

**Classification not point-in-time (deviations register A11).** Sharadar TICKERS is a current snapshot — today's label applied retroactively to a firm that reclassified in 2015 introduces mild look-ahead for that firm around the reclassification event. Industry membership drifts at far lower frequency than any factor signal (reclassification is a multi-year event), so the error is thin and confined to names at sector boundaries. A free point-in-time path exists via EDGAR 10-K header SIC codes (filing-dated, available for the full history); it is deferred rather than unavailable.

### Deliverables

**`data/out/industries_use4.parquet`** — one industry-factor label per stock per signal date: `permaticker`, `signal_date`, `in_estu`, `mcap`, `sharadar_industry` (raw audit column), `industry` (the factor label — one of the 55), `use4_sector`.

**`data/out/industry_weights_use4.parquet`** — per-date industry cap-weight vector: `signal_date`, `industry`, `n_members`, `cap_weight` (industry share of total in-ESTU cap, sums to 1 per date). This is the w-vector for the cap-weighted zero-sum identification constraint; it is materialized here so the cross-sectional regression never re-derives it ad hoc.

### Validation (8-check battery)

1. In-ESTU `UNASSIGNED` rows ≤ 2 at every date
2. All 55 factors have ≥ 1 ESTU member at every date
3. Non-exception factors: median ≥ 15 and min ≥ 8 members; the 4 exceptions reported with their stats
4. Every permaticker carries exactly one factor label (static map)
5. Exactly one row per (permaticker, signal_date)
6. Largest factor ≤ 30% of ESTU cap at every date (reference: 14.1% max)
7. Six mega-cap spot checks: AAPL → Computers & Electronics; JPM → Banks; XOM → Oil Gas & Consumable Fuels; AMZN → Internet & Catalog Retail; NVDA → Semiconductors; PLD → Real Estate
8. Read-back equivalence between disk artifact and in-memory construction

---

## Country factor

### Purpose and economic role

The Country factor (sometimes called the Market or Intercept factor) is the single most important factor in the model by R² contribution. It captures the common return that all US stocks share — the market-level move not explained by industry membership or systematic style tilts. When the S&P 500 drops 3% on a bad macro day, virtually every US stock declines regardless of its size, value, or momentum characteristics; the Country factor absorbs that common component.

The naming convention ("Country" rather than "Market") comes from USE4's global model heritage: in a multi-country model, each country has its own intercept. For a US-only model the distinction is cosmetic, but the structure is exact — every stock's country exposure is identically 1.0, meaning the Country factor return is the regression's intercept for that month.

**The Country factor is not a modeled exposure.** There is nothing to estimate or construct on the exposure side — every stock's country exposure is 1.0 by definition. The real content of this pipeline step is (a) assembling the **anchor** (the monthly regression universe with √mcap weights), and (b) running a **validation cross-sectional regression** that proves the full system (country + 55 industries + 12 styles, with the industry identification constraint) assembles into a well-posed, economically correct regression.

### Relationship to Beta

A common confusion: the Country factor and Beta are distinct despite both relating to the "market." The Country factor is the *return* common to all US stocks — the intercept of the cross-sectional regression for each month. Beta is each stock's *sensitivity* to that common return — the coefficient in the time-series regression of a stock's excess return on the market return. A high-Beta stock earns more from a positive Country factor return than a low-Beta stock, but both enter the cross-sectional regression with identical country exposure (= 1). The Country factor enters every stock equally; Beta gives each stock a differential slope coefficient that magnifies or dampens the market effect.

### Identification constraint and implementation

With a Country column of ones and 55 binary industry columns, the design matrix has a perfect collinearity: the industry columns sum exactly to the country column. The cross-sectional regression is rank-deficient by one. USE4 resolves this with the linear constraint:

```
Σ_j  w_jt · f_ind,j,t  =  0       w_jt = cap weight of industry j in ESTU at t
```

**Implementation via exact reparametrization.** Eliminate the largest-cap-weight industry `e` at each date (numerically safest choice — the largest weight is the most stable divisor). For j ≠ e, the industry factor return is free. The eliminated factor's return is expressed as:

```
f_e = −(Σ_{j≠e} w_j · f_j) / w_e
```

The 55 industry dummy columns transform to 54 reduced columns:

```
D̃_j = D_j − (w_j / w_e) · D_e       for j ≠ e
```

The reduced system `[1 | D̃ | S]` (1 country + 54 reduced industry + 12 style = 67 columns) is solved by WLS with √mcap weights, then `f_e` is recovered from the constraint. This is exact restricted least squares — algebraically identical to Lagrangian and pseudo-inverse approaches, but numerically the most stable because the pivot is always the largest weight. The choice of which industry to eliminate changes nothing but numerical conditioning.

### WLS weights

The regression is weighted by √mcap (not mcap, not equal weights). This is USE4's idiosyncratic variance model: idiosyncratic return variance is approximately proportional to 1/√mcap (larger firms have more analyst coverage, lower information risk, and smaller microstructure noise). Weighting by √mcap down-weights small-cap stocks that contribute more noise per unit of information. The weights are normalized per date to sum to 1, then stored in the anchor deliverable as `w_reg`.

### Validation CSR

The validation runs a constrained WLS regression over every completed month-transition (exposure date t → next signal date t+1) to confirm the full system is well-posed and the Country factor return behaves as USE4 describes. Key checks:

| Check | Gate | USE4 reference |
|---|---|---|
| corr(f_c, monthly benchmark excess return) | > 0.99 | USE4 reports ≈ 0.999 |
| max \|Σ_j w_j · f_ind,j\| over all dates | < 1e-10 | Constraint holds to machine precision |
| Mean weighted cross-sectional R² | > 0.15 | Informational; full distribution reported |
| Max condition number of √v-scaled design | < 1e6 | Well-posed system at every date |
| Dropped factor-dates (null f for a style) | ≤ 9, only early-1999 warm-up | gro, beta, nlb have no scored stocks before warm-up |
| Anchor integrity: Σ w_reg per date | = 1 to 1e-12 | Normalized weights |
| Monthly return-missing share | < 2% | Delistings and RF publication lag |

The check that corr(f_c, benchmark excess return) > 0.99 is the most diagnostic: if the Country factor return does not track the market, something is wrong with either the regression setup, the industry constraint, or the style exposures.

### RF publication lag

The Ken French daily risk-free rate series publishes with a lag of several weeks. Daily excess returns for the most recent weeks are therefore null, meaning some month-end transitions near the data vintage have no usable monthly returns. These transitions are skipped (gated ≤ 3, all within the last 3 calendar months) and picked up on the next data refresh. On the reference run: 2 transitions skipped (2026-04-30 and 2026-05-29 exposure starts), leaving 327 of 329 possible transitions in the validation CSR.

### Output schema

**`data/country_use4.parquet`** (the model anchor):

| Column | Type | Description |
|---|---|---|
| `permaticker` | Int64 | Sharadar permanent ticker ID |
| `signal_date` | Date | End-of-month rebalance date |
| `in_estu` | Bool | Always True — the anchor is the regression universe |
| `mcap` | Float64 | Market cap on signal_date |
| `country` | Float64 | Country exposure — always exactly 1.0 |
| `w_reg` | Float64 | √mcap regression weight, normalized to sum to 1 per date |

**`data/csr_validation_returns.parquet`** (validation artifact, not a model deliverable):

| Column | Type | Description |
|---|---|---|
| `signal_date` | Date | Exposure date t |
| `ret_date` | Date | End of the return month (next signal date) |
| `factor` | String | `country`, 55 industry names, or 12 style names |
| `f` | Float64 | Factor return for the month (null if the column was degenerate) |
| `r2` | Float64 | Weighted cross-sectional R² for that month |
| `n_stocks` | UInt32 | Stocks in that month's regression |

---

## Putting it together

The cross-sectional regression that uses all these exposures runs **after** all factor exposure deliverables are built. Its design matrix has `n_stocks_t` rows and 67 columns: 1 country + 54 reduced-industry (post-reparametrization) + 12 style, plus the 55th industry return recovered from the constraint. The regression runs per month-transition, recovering 68 factor returns from the cross-section of stock returns.

**Build order enforced by the pipeline:**

```
estu_build
  └──► daily_build
         └──► {size, beta, bp, eyld, dyld, lev, liq, gro, mom}   (parallel)
                ├──► resvol         (needs beta)
                ├──► nlb            (needs beta)
                └──► nls            (needs size)
  └──► industries_build             (needs estu only)
  └──► country_build                (needs all of the above — runs last)
```

For data-availability gaps relative to USE4 embedded in any of these steps, see [`data/README.md`](../data/README.md#limitations-relative-to-use4).
