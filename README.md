# USE4-Style Factor Construction with Sharadar

An educational, from-scratch reconstruction of a **USE4-style US equity factor model** built from a Sharadar SF1 subscription (~$49/month via Nasdaq Data Link), Fama–French factors (free), and the published MSCI Barra USE4 methodology and empirical notes.

It is meant for hopeful quants who want something serious and technical on their GitHub, and who are willing to grind through the messy bits: data cleaning, point-in-time issues, debugging, and constantly asking whether the numbers make sense.

> **Disclaimer.** This project is **independent** and **not affiliated with or endorsed by MSCI**. The goal is to *approximate* USE4-style factors and an estimation universe (ESTU) using Sharadar and Fama–French data. It will not match the commercial implementation or its proprietary inputs. Treat it as a training ground, not a drop-in replacement for a production risk system.

---

## How to start

Read the **READMEs** all the way through and take notes. Refer back to them consistently. They are intentionally concise, but filled with valuable information. Understanding what you're getting yourself into before you start will likely be the single most helpful thing you can do for yourself in this repo.

---

## What you'll build

By working through this repo, you'll end up with:

- **A USE4-like estimation universe (ESTU)** for US equities, built from Sharadar data and designed to approximate a broad, investable, liquid universe similar in spirit to MSCI USA IMI.
- **A set of style factors** constructed as close as is reasonable to USE4 definitions given public data: Beta, Size, Momentum, Earnings Yield, Dividend Yield, Residual Volatility, Growth, Book-to-Price, Leverage, Liquidity, Non-linear Size, and Non-linear Beta.
- **~60 GICS-like industry factors** — unit-exposure dummies (every stock gets 0 or 1 to its industry; there is no standardisation step, unlike the style factors). The key challenge is that Sharadar carries SIC codes, Fama–French classifications, and its own sector/industry fields but not GICS directly; building a defensible GICS-like mapping from those fields, and keeping it point-in-time as companies get reclassified, is the real work. Exposures are one-hot membership per (stock, signal date), with explicit rules for conglomerates and names with stale metadata. Industry dummies only matter inside the cross-sectional regression alongside your style exposures — build this after the 12 style factors.
- **Per-factor audits** (in LaTeX) that compare each implementation against its intended economic meaning, check distributions, stability, and correlations, and run simple tests like quintile spreads to catch mistakes.
- **A deviations register** (`docs/deviations_register.tex`) — every place public data forces you off the published USE4 spec, what we chose instead, and how to defend it. Read it before an interview.

If you're new to the space and the data, budget roughly **a month of focused work** to get through ESTU plus a few core factors at a solid level.

---

## Data and environment

### Data sources (you must obtain these yourself)

- **[Sharadar SF1](https://data.nasdaq.com/databases/SF1)** basic-tier datasets (~$49/month via Nasdaq Data Link):
  - Core fundamentals (e.g. SF1/SF3-style tables)
  - Daily prices and corporate actions
  - Any other tables you choose for liquidity, shares, etc.
- **[Fama–French factor datasets](https://mba.tuck.dartmouth.edu/pages/faculty/ken.french/ftp/)** for market/benchmark excess returns where convenient (free, hosted by Dartmouth).

You'll need to be comfortable reading the Sharadar schema docs and mapping fields to what the USE4 methodology expects (market cap, earnings, book value, sales, assets, dividends, and so on).

### Environment

- **Language:** Python only
- **Package management:** [uv](https://github.com/astral-sh/uv)
- **Core libraries:** `polars` for data wrangling, plus the usual numerical/scientific stack

High-level setup:

```bash
uv sync            # install dependencies
uv run python ...  # run scripts, or open notebooks via your IDE
```

Check the `pyproject.toml` / `uv` config in the repo for exact commands once you clone it.

**Hardware.** This is designed to be doable on a reasonably modern laptop. Full-universe daily builds over decades of data will chug, so subset by date or use a smaller ESTU while you iterate.

---

## Repo layout

The repo is organized in **numbered steps** — do them in order.

```
00_data_cleaning/             # STEP 0 — parse and clean raw Sharadar CSVs into parquet
  cleaning_spec.ipynb         #   schema notes, column types, split/currency handling
  cleaning_build.ipynb        #   <- YOU write this

01_estu/                      # STEP 1 — the estimation universe (build this first)
  estu_spec.ipynb             #   design and reasoning; your build target
  estu_audit.tex              #   reference audit (our numbers, for comparison)
  estu_build.ipynb            #   <- YOU write this

02_style_factors/             # STEP 2 — the 12 style factors
  beta/                       #   e.g. beta/, size/, mom/, ...
    beta_spec.ipynb           #   definition, USE4 references, forced deviations
    beta_audit.tex            #   reference audit
    beta_build.ipynb          #   <- YOU write this
  daily_panel/                #   the refactor step: shared returns panel you
                              #   extract after your second time-series factor
  ...

03_industry_factors/          # STEP 3 — 55-factor industry scheme (shipped)
04_country_factor/            # STEP 4 — the market intercept + constraint (shipped; runs last)

common/                       # your shared utilities (you create this as you go)
data/                         # not version-controlled; see data/README.md
docs/                         # factor overview, usage, deviations register
```

Build notebooks are deliberately **not** provided — writing them is the point.
The specs tell you what to build and why; the reference audits tell you what
correct output looked like on our run.

### Status — open to updates

| Step | Contents | Status |
|---|---|---|
| 00 Data cleaning | Sharadar CSV → parquet cleaning pipeline | planned — repo updates when built |
| 01 ESTU | spec + reference audit | **shipped** |
| 02 Style factors | 12 specs + reference audits + daily-panel refactor module | **shipped** |
| 03 Industry factors | spec + 154-atom scheme + reference audit | **shipped** |
| 04 Country factor | spec + reference audit (anchor + validation CSR — runs last) | **shipped** |

Specs and audits ship from a living research pipeline; expect occasional
refinements to thresholds and conventions as the later steps land.

---

## How to proceed

Different people want different amounts of hand-holding. Below are three paths — the labels are a bit tongue-in-cheek. Pick the one matching how much time and pain you're willing to sign up for.

### 1. Fast lane

You fit here if you're not trying to become a polars wizard overnight, you want a respectable factor-model build on your GitHub, and you want to learn to wield AI code assistants effectively without outsourcing the thinking.

1. **Learn the data basics.** Spend time with the Sharadar schema. Understand where fundamentals live (earnings, book value, assets, sales, leverage) and where prices and corporate actions live, including how to get clean, adjusted time series.
2. **Build the ESTU first.** It's the foundation. The market is huge; focus on liquid, investable names rather than every micro-cap zombie. Keep all ESTU logic in one notebook (and ideally one AI chat) so the assistant has context when you discover something is broken. Make sure you understand every filter: why it's there, what it excludes, and what edge cases sneak through.
3. **Then build Beta.** A good first factor: it needs only ESTU plus clean prices and is time-series based (regressing stock excess returns on a market/country factor), so you get used to returns, alignment, and look-ahead issues. Be explicit about windows, frequencies, and what counts as "the market," then check exposure distributions and sanity-check against expectations.
4. **Turn your Beta work into an AI "build factor" skill.** Write a prompt/helper that takes a factor spec (e.g. "Size", "Dividend Yield") and emits a rough draft build notebook. [Claude](https://claude.ai/) (~$20/month) works well for this — it handles long context and can hold the full ESTU + factor spec in one chat. Make it efficient and robust — but treat the output as **deeply untrustworthy** until you beat on it.
5. **Audit everything.** Print dataframes and eyeball rows. Check min/median/max, standard deviations, quintiles, and null counts. Compare against the provided audits. Assume the AI did something stupid you haven't noticed yet.

From there it's rinse and repeat: read a new factor's spec, bootstrap a notebook with your build-factor skill, manually audit and refine, then stress-test against the LaTeX audits. You'll finish with a solid set of factors and a story you can tell in interviews about how you designed, validated, and debugged them.

### 2. Mastery path

You're here if you have time and curiosity, actually want to understand risk models rather than just have one, and are okay getting stuck and unstuck repeatedly.

Before diving in, know or actively learn:

- **What the factors mean.** Why Beta and Country are distinct in USE4, how Size differs from Non-linear Size, what Residual Volatility isolates, how Momentum is defined (and why the most recent month is excluded).
- **How to navigate polars.** Cross-sectional ops, joins, groupby-agg, and especially time-series windows that avoid look-ahead bias.
- **Schemas and limitations.** For Sharadar: what is point-in-time vs not, update lags, missing fields. For Fama–French: what each column represents and how it aligns to your market-return notion.
- **USE4 methodology and priorities.** Read enough of the methodology and empirical notes to understand how descriptors are standardized, how style factors are formed from descriptors, and how the ESTU is supposed to behave (representation, liquidity, stability).

Then:

1. **Start with ESTU and make it airtight.** Use the spec as a starting point, not gospel. Think hard about outliers and edge cases (sudden mcap spikes, illiquid names, weird listings). Goal: you can defend every filter to a skeptical PM.
2. **Build Size first.** Conceptually simple (log market cap plus standardization) but central to many other things, including Non-linear Size. Pay attention to how you define market cap (float-adjusted proxies, currency, timing) and the standardization (cap-weighted mean 0, equal-weighted std 1, consistent with the USE4 style convention).
3. **Then choose your own adventure.** Non-linear Size is a natural follow-up. Dividend Yield is deceptively hard (special dividends, zero-dividend stocks, dividend timing vs price date, survivorship). Earnings Yield, Book-to-Price, and Leverage are all fertile ground for "what exactly do we do with negative or missing values?"

You'll almost certainly disagree with some of the build specs. Good — that's the point. You're not MSCI and you have different goals and risk tolerances. The useful skill is learning when to trust your understanding and when the data humbles you. Use this repo to develop real opinions about what a "good" factor looks like.

### 3. Wing it (but please don't *totally* wing it)

If you ignore everything else here, at least:

- **Build the ESTU first.** Don't build sophisticated style factors on top of a junk universe.
- **Build Beta before Non-linear Beta.** Don't start with the weird non-linear variant before you understand the base factor.
- **Don't start with the hairiest factors.** Get a feel for the mechanics on simpler ones (Size, Beta) before tackling Dividend Yield or complex value/growth composites.

---

## Roadmap

This repo starts with **ESTU and factor exposures** — the cross-sectional building blocks of a USE4-style model — and will grow to cover the full pipeline:

- Use these factor exposures to build a **factor covariance matrix** (something USE4-inspired but tractable on public data).
- Layer on **specific-risk** approximations and simple portfolio risk forecasts.
- Show how to plug this into **basic optimization** and risk attribution.

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