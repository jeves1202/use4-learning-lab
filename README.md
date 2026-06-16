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

Start by reading, in order:

1. **This README** — scope, data sources, repo layout, and build order.
2. **`docs/FACTOR_OVERVIEW.md`** — what each of the 12 style factors, the industry scheme, and the country factor are and why they exist.
3. **`data/README.md`** — the data schema, field mapping, and structural limitations relative to USE4 (analyst estimates, float mcap, preferred equity, GICS).
4. **`docs/USAGE.md`** — conventions and utilities you'll use throughout.

Then build in this order:

5. **`00` — Data cleaning.** Parse raw Sharadar CSVs into parquet. Everything downstream depends on this.
2. **`01` — ESTU.** The estimation universe is the foundation. Don't build factors on top of a junk universe.
3. **`02` — Size, Beta, BP (first three style factors).** Size is conceptually simple and upstream of Non-linear Size. Beta introduces the time-series pattern (rolling regression on excess returns) you'll reuse for Momentum, Residual Volatility, NLB, and NLS. BP is the first fundamentals-based factor — it forces you to wrestle with point-in-time joins and how to handle negatives.
4. **`01.5` — Daily returns panel.** By this point you've written the price-loading and return-computation logic two or three times. Extract it here so the remaining time-series factors load a single parquet instead of reimplementing the same pipeline. Build this after Beta and BP, not before.
5. **`02` — Remaining style factors** (EYLD, DYLD, LEV, LIQ, GRO, MOM, RESVOL, NLS, NLB). Order within this group is flexible; the specs note any inter-factor dependencies.
6. **`03` — Industry factors.** Unit-exposure dummies across the 55-factor scheme. Build after you have the style factors working so you understand the cross-sectional regression context these live in.
7. **`04` — Country factor.** The market intercept with cap-weighted zero-sum industry constraint. Runs last because it depends on both the style and industry factor exposures.

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