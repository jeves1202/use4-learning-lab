# USE4-Style Factor Construction with Sharadar

An educational, from-scratch reconstruction of a **USE4-style US equity factor model** built entirely from public ingredients: basic Sharadar datasets, Fama–French factors, and the published MSCI Barra USE4 methodology and empirical notes.

It is meant for hopeful quants who want something serious and technically honest on their GitHub, and who are willing to grind through the messy bits: data cleaning, point-in-time issues, debugging, and constantly asking whether the numbers make sense.

> **Disclaimer.** This project is **independent** and **not affiliated with or endorsed by MSCI**. The goal is to *approximate* USE4-style factors and an estimation universe (ESTU) using public data. It will not match the commercial implementation or its proprietary inputs. Treat it as a training ground, not a drop-in replacement for a production risk system.

---

## What you'll build

By working through this repo, you'll end up with:

- **A USE4-like estimation universe (ESTU)** for US equities, built from Sharadar data and designed to approximate a broad, investable, liquid universe similar in spirit to MSCI USA IMI.
- **A set of style factors** constructed as close as is reasonable to USE4 definitions given public data: Beta, Size, Momentum, Earnings Yield, Dividend Yield, Residual Volatility, Growth, Book-to-Price, Leverage, Liquidity, Non-linear Size, and Non-linear Beta.
- **Per-factor audits** (in LaTeX) that compare each implementation against its intended economic meaning, check distributions, stability, and correlations, and run simple tests like quintile spreads to catch mistakes.

If you're new to the space and the data, budget roughly **a month of focused work** to get through ESTU plus a few core factors at a solid level.

---

## Data and environment

### Data sources (you must obtain these yourself)

- **Sharadar** basic-tier datasets:
  - Core fundamentals (e.g. SF1/SF3-style tables)
  - Daily prices and corporate actions
  - Any other tables you choose for liquidity, shares, etc.
- **Fama–French factor datasets** for market/benchmark excess returns where convenient.

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

The repo is organized by **ESTU** and **factor symbols**, each in its own folder.

```
estu/
  estu_spec.ipynb     # design and reasoning for the estimation universe
  estu_build.ipynb    # the actual construction pipeline
  estu_audit.tex      # coverage, liquidity, sector weights, turnover, etc.

factors/<factor_name>/        # e.g. beta/, size/, mom/, ...
  <factor>_spec.ipynb   # definition, USE4 references, deviations forced by public data
  <factor>_build.ipynb  # construction from Sharadar + ESTU
  <factor>_audit.tex    # distributions, correlations, quintile returns, bug checks

common/               # (or src/...) shared utilities:
  - loading and cleaning Sharadar tables
  - handling point-in-time and look-ahead issues
  - ESTU filters (market-cap, liquidity, listing status)
  - standardization, winsorization, and other shared transformations

data/                 # not version-controlled; see data/README.md for:
  - what raw files are expected where
  - filename conventions
  - how to point notebooks at your local data

docs/                 # detailed notes, factor overviews, extra commentary
```

---

## How to work through the repo

Different people want different amounts of hand-holding. Below are three paths — the labels are a bit tongue-in-cheek. Pick the one matching how much time and pain you're willing to sign up for.

### 1. Fast lane

You fit here if you're not trying to become a polars wizard overnight, you want a respectable factor-model build on your GitHub, and you want to learn to wield AI code assistants effectively without outsourcing the thinking.

1. **Learn the data basics.** Spend time with the Sharadar schema. Understand where fundamentals live (earnings, book value, assets, sales, leverage) and where prices and corporate actions live, including how to get clean, adjusted time series.
2. **Build the ESTU first.** It's the foundation. The market is huge; focus on liquid, investable names rather than every micro-cap zombie. Keep all ESTU logic in one notebook (and ideally one AI chat) so the assistant has context when you discover something is broken. Make sure you understand every filter: why it's there, what it excludes, and what edge cases sneak through.
3. **Then build Beta.** A good first factor: it needs only ESTU plus clean prices and is time-series based (regressing stock excess returns on a market/country factor), so you get used to returns, alignment, and look-ahead issues. Be explicit about windows, frequencies, and what counts as "the market," then check exposure distributions and sanity-check against expectations.
4. **Turn your Beta work into an AI "build factor" skill.** Write a prompt/helper that takes a factor spec (e.g. "Size", "Dividend Yield") and emits a rough draft build notebook. Make it efficient and robust — but treat the output as **deeply untrustworthy** until you beat on it.
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

## Roadmap and related repos

This repo focuses on **ESTU and factor exposures** — the cross-sectional building blocks of a USE4-style model. Separate repos will build on it to:

- Use these factor exposures to build a **factor covariance matrix** (something USE4-inspired but tractable on public data).
- Layer on **specific-risk** approximations and simple portfolio risk forecasts.
- Show how to plug this into **basic optimization** and risk attribution.

If you care about turning these factors into an actual risk model, stay tuned.

---

## License & attribution

Educational and independent. Not affiliated with or endorsed by MSCI. Sharadar and Fama–French data are subject to their respective licenses and terms — obtain and use them accordingly.