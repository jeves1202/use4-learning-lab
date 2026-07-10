# USE4-Style Factor Risk Model with Sharadar

An educational, from-scratch reconstruction of a **USE4-style US equity factor risk model** — from raw vendor CSVs all the way to a factor covariance matrix, specific-risk forecasts, and portfolio risk decomposition — built from a Sharadar SF1 subscription (~$49/month via Nasdaq Data Link), Fama–French factors (free), and the published MSCI Barra USE4 methodology and empirical notes.

It is meant for hopeful quants who want something serious and technical on their GitHub, and who are willing to grind through the messy bits: data cleaning, point-in-time issues, debugging, and constantly asking whether the numbers make sense.

> **Disclaimer.** This project is **independent** and **not affiliated with or endorsed by MSCI**. The goal is to *approximate* USE4-style factors and an estimation universe (ESTU) using Sharadar and Fama–French data. It will not match the commercial implementation or its proprietary inputs. Treat it as a training ground, not a drop-in replacement for a production risk system.

---

## Current State

The only shipped contruction guides (specs) are stages 00, 01, 01.5, and `size` and `beta` within 02. The textbook is fully shipped. The rest of the build guides will be published shortly. All shipped elements are in beta development, and any feedback is greatly appreciated. Direct all feedback to:

```bash
jakobseves@gmail.com
```

---

## How this repo works

Each numbered step ships a spec and withholds the build; the textbook is the reference:

- **`docs/textbook/`** — the reference. Most steps have a chapter (the estimation universe, each style factor, industries, the CSR, factor covariance, specific risk, risk decomposition) whose **Measured** boxes carry the numbers a correct build reproduces — in shape, not vintage — alongside the full theory. It's your answer key.
- **`*_spec.ipynb`** — a markdown-only spec: USE4 quotes, every judgment call the PDFs don't settle (with a chosen default), validation targets, and known pitfalls. This is your build target.
- **`*_build.ipynb`** — deliberately **not provided**. Writing it is the point. The spec tells you what to build and why.

Read the READMEs all the way through before you start and refer back to them. They are intentionally concise but dense — understanding what you're getting into is the single most helpful thing you can do for yourself here.

**Note** I suggest building your factors in `.ipynb` format, but if you intend to run the entire pipeline at once, then switiching to `.py` will be essential for most hardware.

---

## What you'll build

Working front to back, you end up with a complete risk model:

- **An estimation universe (ESTU)** for US equities — broad, investable, liquid, similar in spirit to MSCI USA IMI.
- **12 style factors** built as close to USE4 definitions as public data allows: Beta, Size, Momentum, Earnings Yield, Dividend Yield, Residual Volatility, Growth, Book-to-Price, Leverage, Liquidity, Non-linear Size, Non-linear Beta.
- **~55 industry factors** — unit-exposure dummies you design yourself from Sharadar's ~150 classification atoms, since GICS isn't available. Keeping the scheme defensible and point-in-time is the real work.
- **The country factor anchor** — the market intercept, plus the cap-weighted zero-sum industry constraint that identifies it.
- **The cross-sectional regression (CSR)** — the engine that turns exposures into monthly factor returns and specific returns, then runs again per trading day to feed everything downstream.
- **A factor covariance matrix** — EWMA volatilities and correlations, Newey–West correction, Monte-Carlo eigenfactor de-biasing, and a volatility-regime multiplier.
- **Specific risk** — a five-layer per-stock idiosyncratic volatility model (time-series, structural, coverage blend, Bayesian shrinkage, regime multiplier).
- **Portfolio risk decomposition** — σ²(portfolio) = wᵀXFXᵀw + wᵀΔw, with Euler attribution by factor group and active risk versus the cap-weighted market.

Every step ends with an explicit PASS/FAIL validation battery, and the textbook chapter documents the reference numbers you're trying to reproduce in shape (not in vintage).

If you're new to the space and the data, budget roughly **a month of focused work** to get through ESTU plus a few core factors at a solid level. The full risk model is a bigger commitment — but it's built one bounded step at a time.

---

## Data and environment

### Data sources (you must obtain these yourself)

- **[Sharadar SF1](https://data.nasdaq.com/databases/SF1)** basic-tier datasets (~$49/month via Nasdaq Data Link):
  - Core fundamentals (e.g. SF1/SF3-style tables)
  - Daily prices and corporate actions
  - Any other tables you choose for liquidity, shares, etc.
- **[Fama–French factor datasets](https://mba.tuck.dartmouth.edu/pages/faculty/ken.french/)** for market/benchmark excess returns where convenient (free, hosted by Dartmouth).

You'll need to be comfortable reading the Sharadar schema docs and mapping fields to what the USE4 methodology expects (market cap, earnings, book value, sales, assets, dividends, and so on). `data/README.md` does most of this for you.

### Environment

- **Language:** Python only
- **Package management:** [uv](https://github.com/astral-sh/uv)
- **Core libraries:** `polars` for data wrangling, plus the usual numerical/scientific stack

```bash
uv sync                   # install dependencies
uv run python ...         # run scripts, or open notebooks via your IDE
uv add [dependency_name]  # adds new dependencies, and will be reflected in uv.lock and pyproject.toml
```

**Hardware.** Doable on a reasonably modern computer/laptop. Full-universe daily builds over decades of data, so it is useful to consider the tradeoffs between a multi-threaded approach (when possible) -- which is faster but has higher RAM requirements -- and a purely sequential approach. The best solution will likley be a hybrid of both.

---

## Repo layout

Numbered steps — do them in order.

```
00_data_cleaning/             # STEP 0 — raw Sharadar CSVs → cleaned parquet
  cleaning_spec.ipynb         #   principles + your build target
  cleaning_build.ipynb        #   <- YOU write this
01_estu/                      # STEP 1 — the estimation universe
  estu_spec.ipynb             #   the spec; your build target
  estu_build.ipynb            #   <- YOU write this
01.5_daily/                   # STEP 1.5 — shared daily returns panel (feeds the time-series factors)
02_style_factors/             # STEP 2 — the 12 style factors
  beta/                       #   beta/, size/, mom/, ... one dir per factor
    beta_spec.ipynb
    beta_build.ipynb          #   <- YOU write this
  daily_panel/                #   the refactor module the 01.5 step formalizes
03_industry_factors/          # STEP 3 — industry factors (you design the scheme)
04_country_factor/            # STEP 4 — market intercept anchor + validation CSR
05_csr/                       # STEP 5 — the cross-sectional regression
  csr_spec.ipynb              #   monthly production CSR
  daily_csr_spec.ipynb        #   daily sibling — Layer 0 for steps 06 and 07
06_fcov/                      # STEP 6 — factor covariance forecast
07_specific_risk/             # STEP 7 — specific (idiosyncratic) risk
08_risk_decomp/               # STEP 8 — portfolio risk decomposition (runs last)
data/                         # not version-controlled; see data/README.md
docs/                         # factor overview + usage conventions
  textbook/                   # the reference: one chapter per stage (.tex + PDFs)
```

### The steps

| Step | What it gives you |
|---|---|
| 00 Data cleaning | cleaning spec (principles) |
| 01 ESTU | spec + textbook chapter |
| 01.5 Daily panel | spec |
| 02 Style factors | 12 specs + textbook chapters |
| 03 Industry factors | spec + textbook chapter (scheme design is yours) |
| 04 Country factor | spec (anchor + validation CSR) + textbook chapter (CSR) |
| 05 CSR | monthly + daily specs + textbook chapter |
| 06 Factor covariance | spec + textbook chapter |
| 07 Specific risk | spec + textbook chapter |
| 08 Risk decomposition | spec + textbook chapter |

Every step ships a spec (your build target) and, for most, a textbook chapter (your reference); the build notebooks are yours to write. The textbook's **Measured** boxes are drawn from a personal build run — treat them as targets to reproduce in shape, not vintage.

---

## How to proceed

Read first, in order: **this README** → **`docs/USAGE.md`** (the per-step loop and conventions).

Then build:

1. **`00` — Data cleaning.** Raw CSVs → typed, partitioned parquet. Everything downstream reads from it.
2. **`01` — ESTU.** The foundation. Don't build factors on a junk universe.
3. **`01.5` — Daily returns panel.** The shared price/return plumbing every time-series factor reads — excess returns and the ESTU cap-weighted benchmark. Build it here, once, so each style factor just loads a parquet. *(Suggestion, not a requirement: if you'd rather feel the return machinery before abstracting it, build your first time-series factor — Beta — with the plumbing inline and extract the panel afterward; that's how the reference build grew. Either way, the style specs assume the panel exists.)*
4. **`02` — Size, Beta, BTOP.** One spot fundamental, one time-series factor, one point-in-time join — you touch every fundamental mechanic you'll need for the style factors.
5. **`02` — remaining style factors** (EYLD, DYLD, LEV, LIQ, GRO, MOM, RESVOL, NLS, NLB). Order is flexible; respect `beta → {resvol, nlb}` and `size → nls`.
6. **`03` — Industry factors.** You design the scheme; the spec explains the criteria.
7. **`04` — Country factor.** The anchor (unit exposure + √mcap weights) plus a validation CSR that proves the whole system is well-posed.
8. **`05` — The cross-sectional regression.** First the monthly production CSR (factor returns + specific returns), then the daily sibling — same engine run per trading day, with month-end exposures held fixed. The daily outputs are Layer 0 for both of the next two steps.
9. **`06` — Factor covariance.** EWMA → Newey–West → eigenfactor → volatility regime, forecast at each month-end.
10. **`07` — Specific risk.** Five layers from daily CSR residuals to a per-stock monthly σ. Consumes step 05's daily output directly — it does not depend on 06.
11. **`08` — Risk decomposition.** Composes X, F, and Δ into portfolio risk and attribution. Runs last.

**A note on shared plumbing.** The risk-model stages (05–08) load the same exposure deliverables, the same daily panel, and the same month-owned return mapping over and over. The specs keep each build self-contained so every step stands on its own, but once you start step 05 it is very much worth extracting a small common library for the shared loads and kernels — it saves real compute and real debugging. That's what I did in my own build.

---

## Roadmap

The model is specified end to end: data cleaning → ESTU → exposures → CSR → covariance → specific risk → decomposition. The natural next layer is:

- **Portfolio optimization** — putting the risk model to work in a basic optimizer, on top of step 08's decomposition machinery.

---

## References

[1] Menchero, J., Orr, D.J., Wang, J. (2011). "The Barra US Equity Model (USE4)
    Methodology Notes." MSCI Research, August 2011.
    URL: https://www.top1000funds.com/wp-content/uploads/2011/09/USE4_Methodology_Notes_August_2011.pdf
    (Primary source — all mathematical formulas and empirical results)

[2] MSCI (2011). "Barra US Equity Model (USE4) Empirical Notes."
    URL: http://index.cslt.org/mediawiki/images/4/47/MSCI-USE4-201109.pdf

[3] MSCI Research (2011). "MSCI Launches New Barra Equity Models." Press release,
    September 13, 2011.
    URL: https://ir.msci.com/news-releases/news-release-details/msci-launches-new-barra-equity-models

[4] Menchero, J. (2011). MSCI USE4 Presentation to QWAFAFEW Boston, April 2011.
    URL: http://boston.qwafafew.org/wp-content/uploads/sites/4/2017/01/MSCI-4_26_2011.pdf

[5] Menchero, J., Orr, D.J., Wang, J. (2011). "The Barra US Equity Model (USE4)
    Methodology Notes." Semantic Scholar.
    URL: https://www.semanticscholar.org/paper/The-Barra-US-Equity-Model-(-USE-4-)-Methodology-Menchero-Orr/4fc8d22a4d9e22427be83984c5420...

[6] MSCI (2011). "Barra US Equity Model (USE4) Key Features Datasheet."
    URL: https://www.msci.com/documents/10199/242721/Barra_US_Equity_Model_USE4.pdf

---

## License & attribution

Educational and independent. Not affiliated with or endorsed by MSCI. Sharadar and Fama–French data are subject to their respective licenses and terms — obtain and use them accordingly.
