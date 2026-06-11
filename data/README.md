# Sharadar Data Guide

This file documents every raw dataset in the Sharadar SF1 bundle, with schemas, point-in-time rules, and field-level notes oriented toward building USE4-style factor exposures. Raw CSVs live at:

```
data/raw/SHARADAR_*.csv        # relative to the repo root; not version-controlled
```

Source: [Sharadar SF1 on Nasdaq Data Link](https://data.nasdaq.com/databases/SF1) (~$49/month).

---

## Dataset inventory

| File | Size | Purpose |
|---|---|---|
| `SHARADAR_SF1.csv` | ~2.4 GB | Core fundamentals — income statement, balance sheet, cash flow, derived ratios |
| `SHARADAR_SEP.csv` | ~3.2 GB | Daily equity prices (split-adjusted and fully-adjusted) |
| `SHARADAR_DAILY.csv` | ~2.5 GB | Daily snapshot of market-cap and valuation ratios, tied to latest filing |
| `SHARADAR_TICKERS.csv` | ~20 MB | Ticker metadata: exchange, sector, SIC, size bucket, date ranges |
| `SHARADAR_ACTIONS.csv` | ~46 MB | Corporate actions: splits, dividends, mergers, delistings, ticker changes |
| `SHARADAR_EVENTS.csv` | ~52 MB | 8-K event codes by ticker and date |
| `SHARADAR_SF2.csv` | ~1.7 GB | Insider transactions (Form 4 filings) |
| `SHARADAR_SF3.csv` | ~2.8 GB | 13-F institutional holdings, per ticker per investor |
| `SHARADAR_SF3A.csv` | ~91 MB | Institutional holdings summarised by ticker (quarterly) |
| `SHARADAR_SF3B.csv` | ~43 MB | Institutional holdings summarised by investor (quarterly) |
| `SHARADAR_SFP.csv` | ~1.1 GB | Prices for funds and ETFs (same schema as SEP) |
| `SHARADAR_SP500.csv` | ~3 MB | Historical S&P 500 constituent additions and removals |
| `SHARADAR_INDICATORS.csv` | ~80 KB | Official field descriptions for every table (good first stop) |

For this project the tables you will actually load frequently are **SF1, SEP, DAILY, TICKERS, ACTIONS**, and occasionally **SP500** and **SF3A**. **Indicators** is a helpful Source of Truth.

---

## 1. SHARADAR_SF1 — Core fundamentals

The main table. Each row is one `(ticker, dimension, calendardate)` triple.

### Primary-key columns

| Column | Type | Description |
|---|---|---|
| `ticker` | str | Current ticker symbol. May change over time; see ACTIONS for history. |
| `dimension` | str | Which view of the data — see dimension reference below. |
| `calendardate` | date | Normalised fiscal period end (quarter-end for Q/T dims, year-end for Y dims). Use this for cross-sectional joins. |
| `datekey` | date | **Point-in-time availability date.** For AR* dims: the SEC filing date. For MR* dims: equals `reportperiod`. |
| `reportperiod` | date | Raw end-of-period date from the filing. |
| `lastupdated` | date | Last time Sharadar touched this row. |
| `fiscalperiod` | str | Human-readable period label, e.g. `2024-Q3` or `2024-FY`. |

### The dimension system

This is the single most important thing to understand in SF1.

| Code | Meaning |
|---|---|
| `ARQ` | **As-reported quarterly** — one row per fiscal quarter, dated at the SEC filing date. No restatements applied. |
| `ART` | **As-reported trailing-twelve-months** — four quarters summed, dated at the filing date. |
| `ARY` | **As-reported annual** — one row per fiscal year, dated at the filing date. |
| `MRQ` | **Most-recent quarterly** — like ARQ but restatements included; `datekey = reportperiod`. |
| `MRT` | **Most-recent TTM** — like ART with restatements; `datekey = reportperiod`. |
| `MRY` | **Most-recent annual** — like ARY with restatements; `datekey = reportperiod`. |

**Rule of thumb for factor construction:** always use `AR*` dimensions when you care about point-in-time correctness (i.e., almost always). The `MR*` dims are useful for research where you want the most accurate numbers, but they introduce look-ahead because `datekey` is the period end, not the filing date.

To avoid look-ahead: filter `datekey <= your_signal_date`, then for each ticker take the row with the maximum `datekey`.

### Income statement fields

| Column | Title | Unit |
|---|---|---|
| `revenue` | Revenues | currency |
| `cor` | Cost of Revenue | currency |
| `gp` | Gross Profit | currency |
| `opex` | Operating Expenses (excl. CoR) | currency |
| `sgna` | SG&A | currency |
| `rnd` | R&D Expense | currency |
| `opinc` | Operating Income | currency |
| `intexp` | Interest Expense | currency |
| `ebt` | Earnings Before Tax | currency |
| `taxexp` | Income Tax Expense | currency |
| `consolinc` | Consolidated Income (pre-NCI) | currency |
| `netincnci` | Net Income to Non-Controlling Interests | currency |
| `netinc` | Net Income (to parent) | currency |
| `prefdivis` | Preferred Dividends | currency |
| `netinccmn` | Net Income to Common Stockholders | currency |
| `netincdis` | Net Income from Discontinued Ops | currency |
| `eps` | EPS (Basic, as reported) | currency/share |
| `epsdil` | EPS (Diluted) | currency/share |
| `shareswa` | Weighted-avg shares (basic) | units |
| `shareswadil` | Weighted-avg shares (diluted) | units |
| `dps` | Dividends per Share | USD/share |
| `ebit` | EBIT | currency |
| `ebitda` | EBITDA | currency |

### Cash flow statement fields

| Column | Title | Unit |
|---|---|---|
| `ncfo` | Net Cash from Operations | currency |
| `ncfi` | Net Cash from Investing | currency |
| `ncff` | Net Cash from Financing | currency |
| `ncf` | Net Change in Cash | currency |
| `capex` | Capital Expenditure | currency |
| `ncfbus` | CF from Business Acquisitions/Disposals | currency |
| `ncfinv` | CF from Investment Acquisitions/Disposals | currency |
| `ncfdebt` | Debt Issuance / Repayment | currency |
| `ncfcommon` | Equity Issuance / Repurchases | currency |
| `ncfdiv` | Dividends Paid | currency |
| `ncfx` | FX Effect on Cash | currency |
| `depamor` | Depreciation & Amortisation | currency |
| `sbcomp` | Stock-Based Compensation | currency |

### Balance sheet fields

| Column | Title | Unit |
|---|---|---|
| `assets` | Total Assets | currency |
| `assetsc` | Current Assets | currency |
| `assetsnc` | Non-Current Assets | currency |
| `cashneq` | Cash & Equivalents | currency |
| `investments` | Total Investments | currency |
| `investmentsc` | Current Investments | currency |
| `investmentsnc` | Non-Current Investments | currency |
| `receivables` | Trade & Non-Trade Receivables | currency |
| `inventory` | Inventory | currency |
| `ppnenet` | PP&E (net) | currency |
| `intangibles` | Goodwill & Intangibles | currency |
| `taxassets` | Tax Assets | currency |
| `liabilities` | Total Liabilities | currency |
| `liabilitiesc` | Current Liabilities | currency |
| `liabilitiesnc` | Non-Current Liabilities | currency |
| `debt` | Total Debt (current + non-current) | currency |
| `debtc` | Current Debt | currency |
| `debtnc` | Non-Current Debt | currency |
| `payables` | Trade & Non-Trade Payables | currency |
| `deferredrev` | Deferred Revenue | currency |
| `deposits` | Deposit Liabilities (banks/insurance) | currency |
| `taxliabilities` | Tax Liabilities | currency |
| `equity` | Shareholders Equity (to parent) | currency |
| `retearn` | Retained Earnings / Accumulated Deficit | currency |
| `accoci` | Accumulated Other Comprehensive Income | currency |

### Derived metrics and ratios

| Column | Title | Unit | Notes |
|---|---|---|---|
| `marketcap` | Market Cap | USD | `sharesbas × price × sharefactor` — as of `datekey` |
| `ev` | Enterprise Value | USD | `marketcap + debtusd - cashneq_usd` |
| `price` | Share Price (split-adjusted close) | USD/share | As of `datekey` |
| `sharesbas` | Shares Outstanding (basic) | units | Cover page figure |
| `sharefactor` | Share Factor | ratio | ADR adjustment or dual-class EPS normalisation |
| `bvps` | Book Value per Share | currency/share | `equity / shareswa × sharefactor` |
| `tbvps` | Tangible BV per Share | currency/share | `tangibles / shareswa × sharefactor` |
| `tangibles` | Tangible Assets | currency | `assets - intangibles` |
| `invcap` | Invested Capital | currency | `debt + assets - intangibles - cashneq - liabilitiesc` |
| `invcapavg` | Avg Invested Capital | currency | Average of current and prior period `invcap` |
| `equityavg` | Avg Equity | currency | Average for ROE calculation |
| `assetsavg` | Avg Assets | currency | Average for ROA calculation |
| `workingcapital` | Working Capital | currency | `assetsc - liabilitiesc` |
| `fcf` | Free Cash Flow | currency | `ncfo - capex` |
| `fcfps` | FCF per Share | currency/share | |
| `sps` | Sales per Share | USD/share | |
| `roe` | Return on Avg Equity | ratio | |
| `roa` | Return on Avg Assets | ratio | |
| `roic` | Return on Invested Capital | ratio | `ebit / invcapavg` |
| `ros` | Return on Sales | ratio | `ebit / revenue` |
| `grossmargin` | Gross Margin | ratio | |
| `netmargin` | Net Margin | ratio | |
| `ebitdamargin` | EBITDA Margin | ratio | |
| `assetturnover` | Asset Turnover | ratio | |
| `currentratio` | Current Ratio | ratio | |
| `de` | Debt/Equity | ratio | `liabilities / equity` |
| `pb` | Price/Book | ratio | |
| `pe` | P/E (Damodaran: `marketcap / netinccmnusd`) | ratio | |
| `pe1` | P/E (alt: `price / epsusd`) | ratio | |
| `ps` | P/S (Damodaran) | ratio | |
| `ps1` | P/S (alt: `price / sps`) | ratio | |
| `evebit` | EV/EBIT | ratio | |
| `evebitda` | EV/EBITDA | ratio | |
| `divyield` | Dividend Yield | ratio | Trailing 12-month DPS from ACTIONS |
| `payoutratio` | Payout Ratio | ratio | `dps / epsusd` |

### USD-converted fields

For non-US companies. Sharadar stores local-currency figures in the base columns and appends `usd` for the USD equivalent.

| Column | Source |
|---|---|
| `fxusd` | FX rate used for conversion |
| `equityusd` | `equity × fxusd` |
| `revenueusd` | `revenue × fxusd` |
| `netinccmnusd` | `netinccmn × fxusd` |
| `cashnequsd` | `cashneq × fxusd` |
| `debtusd` | `debt × fxusd` |
| `ebitusd` | `ebit × fxusd` |
| `ebitdausd` | `ebitda × fxusd` |
| `epsusd` | `eps × fxusd` |

### Point-in-time loading pattern (Polars)

```python
import polars as pl

sf1 = pl.scan_csv("...SHARADAR_SF1.csv", try_parse_dates=True)

# point-in-time snapshot as of a given date
def sf1_as_of(date: str, dim: str = "ART") -> pl.DataFrame:
    return (
        sf1
        .filter(pl.col("dimension") == dim)
        .filter(pl.col("datekey") <= pl.lit(date).str.to_date())
        .sort("datekey")
        .group_by("ticker")
        .last()  # most recently filed row per ticker
        .collect()
    )
```

---

## 2. SHARADAR_SEP — Equity prices

Daily OHLCV for every US equity. This is your primary source for returns, momentum, beta, and liquidity.

| Column | Description |
|---|---|
| `ticker` | Ticker symbol |
| `date` | Price date |
| `open` | Open (split-adjusted) |
| `high` | High (split-adjusted) |
| `low` | Low (split-adjusted) |
| `close` | Close (split-adjusted, **not** dividend-adjusted) |
| `volume` | Volume (split-adjusted) |
| `closeadj` | Close adjusted for splits **and** dividends and spinoffs — use this for total-return calculations |
| `closeunadj` | Raw unadjusted close — use for reconstructing shares outstanding or absolute-price calculations |
| `lastupdated` | Last update date |

**Split-adjusted vs dividend-adjusted:** `close` is the right price to show on a chart. `closeadj` gives the return series you need for factor construction (momentum, beta, residual vol). When in doubt, use `closeadj` for returns.

### Loading returns (Polars)

```python
sep = pl.scan_csv("...SHARADAR_SEP.csv", try_parse_dates=True)

returns = (
    sep
    .select(["ticker", "date", "closeadj"])
    .sort(["ticker", "date"])
    .with_columns(
        pl.col("closeadj")
          .pct_change()
          .over("ticker")
          .alias("ret")
    )
)
```

---

## 3. SHARADAR_DAILY — Daily fundamental metrics

A daily time series of market-cap and valuation ratios derived by combining the latest SF1 filing with the current price. Much smaller than SF1 × SEP and convenient for daily market-cap lookups.

| Column | Description |
|---|---|
| `ticker` | Ticker |
| `date` | Date |
| `lastupdated` | Last updated |
| `marketcap` | Market cap (USD thousands — **check units in your version**) |
| `ev` | Enterprise value |
| `evebit` | EV/EBIT |
| `evebitda` | EV/EBITDA |
| `pb` | Price/Book |
| `pe` | P/E (Damodaran) |
| `ps` | P/S (Damodaran) |

**Important:** The ratios in DAILY are constructed from the most recently available filing as of each date. They are point-in-time safe in the sense that they use filed data, but there is a subtlety: the denominator (fundamental) does not change between filings even as the price moves daily.

Use DAILY as a quick source of daily market-cap for ESTU construction and liquidity filters. For more control, compute `marketcap` yourself from TICKERS + SEP + SF1.

---

## 4. SHARADAR_TICKERS — Ticker metadata

One row per ticker. Essential for ESTU construction — exchange, sector, size bucket, listing dates, delisting flag.

| Column | Description |
|---|---|
| `permaticker` | Permanent numeric ID — stable across ticker changes |
| `ticker` | Current ticker |
| `name` | Company name |
| `exchange` | `NYSE`, `NASDAQ`, `NYSEMKT`, `OTC`, etc. |
| `isdelisted` | `Y` / `N` |
| `category` | `Domestic Common Stock`, `ADR`, `ETF`, `Fund`, `REIT`, etc. |
| `cusips` | CUSIP(s) |
| `siccode` | SIC code (4-digit) |
| `sicsector` | Broad SIC sector description |
| `sicindustry` | SIC industry description |
| `figi` | Composite FIGI |
| `famaindustry` | Fama–French industry classification |
| `sector` | Broad sector (Healthcare, Tech, …) |
| `industry` | More granular industry |
| `scalemarketcap` | Size bucket: `1 - Nano` through `6 - Mega` |
| `scalerevenue` | Revenue size bucket (same scale) |
| `relatedtickers` | Related ticker symbols |
| `currency` | Reporting currency |
| `location` | Country / State |
| `lastupdated` | |
| `firstadded` | Date first in database |
| `firstpricedate` | First available price date |
| `lastpricedate` | Last available price date |
| `firstquarter` | First available quarter in SF1 |
| `lastquarter` | Last available quarter in SF1 |
| `secfilings` | URL to SEC EDGAR filings |
| `companysite` | Company website |

**ESTU filter notes:**
- Start with `category = "Domestic Common Stock"` to exclude ETFs, funds, ADRs, preferred shares, etc.
- `exchange` in `{NYSE, NASDAQ, NYSEMKT}` to exclude OTC names.
- `isdelisted = "N"` only if you want the live universe — but for backtests you should **include delisted names** and use `lastpricedate` to determine when they exit your universe. Excluding them creates survivorship bias.
- `scalemarketcap` is a static snapshot (current); for dynamic size filtering during a backtest use DAILY.marketcap.

---

## 5. SHARADAR_ACTIONS — Corporate actions

Event log for splits, dividends, mergers, delistings, spinoffs, and ticker changes.

| Column | Description |
|---|---|
| `date` | Event date |
| `action` | Action type (see below) |
| `ticker` | Primary ticker |
| `name` | Company name |
| `value` | Action-specific value (e.g., split ratio, dividend amount) |
| `contraticker` | Counterpart ticker (e.g., acquirer, spinoff parent) |
| `contraname` | Counterpart name |

### Action types

| Code | Meaning |
|---|---|
| `split` | Stock split or stock dividend — `value` is new-shares-per-old |
| `dividend` | Cash dividend — `value` is dividend per share |
| `spinoffdividend` | Spinoff accompanied by dividend |
| `spinoff` | Spinoff — `value` is spinoff ratio |
| `merger from` | This ticker was merged into `contraticker` |
| `merger to` | This ticker absorbed `contraticker` |
| `acquisition of` | This ticker was acquired |
| `acquisition by` | This ticker acquired `contraticker` |
| `delisted` | Removed from exchange |
| `listed` | Newly listed |
| `tickerchangefrom` | Old ticker (now `contraticker`) |
| `tickerchangeto` | New ticker (now `contraticker`) |
| `bankruptcyliquidation` | Bankruptcy / liquidation |

**Factor construction use:** critical for tracking dividend payments when computing your own dividend yield without relying on the pre-computed `divyield` in SF1. Also important for correctly attributing returns around spinoffs and mergers.

---

## 6. SHARADAR_SP500 — S&P 500 constituents

Historical log of S&P 500 additions and removals.

| Column | Description |
|---|---|
| `date` | Event date |
| `action` | `added`, `removed`, or `current` |
| `ticker` | Ticker |
| `name` | Company name |
| `contraticker` | Replaced by / replaced (for swaps) |
| `contraname` | |
| `note` | Free-text notes |

Useful for constructing a historical S&P 500 universe (e.g., as a starting point for a broad ESTU, or as a benchmark universe for coverage checks). `action = "current"` rows give the current snapshot.

---

## 7. SHARADAR_EVENTS — Material corporate events (8-K)

One row per (ticker, date), with a pipe-delimited list of SEC 8-K event codes.

| Column | Description |
|---|---|
| `ticker` | Ticker |
| `date` | Filing date |
| `eventcodes` | Pipe-separated 8-K item codes, e.g. `22|91` |

Common event codes relevant to factor construction:

| Code | Meaning |
|---|---|
| `22` | Results of Operations and Financial Condition (earnings releases) |
| `21` | Completion of Acquisition or Disposition of Assets |
| `51` | Changes in Control of Registrant |
| `52` | Departure / Election of Directors or Officers |
| `91` | Financial Statements and Exhibits |
| `13` | Bankruptcy or Receivership |
| `31` | Notice of Delisting or Listing Non-Compliance |

---

## 8. SHARADAR_SF2 — Insider transactions (Form 4)

Each row is one insider transaction filed on Form 4.

| Column | Description |
|---|---|
| `ticker` | Ticker |
| `filingdate` | SEC filing date |
| `formtype` | `3`, `4`, or `5` |
| `issuername` | Company |
| `ownername` | Insider name |
| `officertitle` | Role (if officer) |
| `isdirector` / `isofficer` / `istenpercentowner` | Flags |
| `transactiondate` | Date of the actual trade |
| `securityadcode` | `A` = acquired, `D` = disposed |
| `transactioncode` | `P` = open-market purchase, `S` = sale, `G` = gift, etc. |
| `sharesownedbeforetransaction` | |
| `transactionshares` | Shares bought/sold |
| `sharesownedfollowingtransaction` | |
| `transactionpricepershare` | Price |
| `transactionvalue` | Total transaction value |
| `securitytitle` | E.g. `Common Stock`, `Options` |
| `directorindirect` | `D` = direct, `I` = indirect |
| `dateexercisable` / `priceexercisable` / `expirationdate` | For options |

---

## 9. SHARADAR_SF3 / SF3A / SF3B — Institutional holdings (13-F)

**SF3** — per ticker per investor, quarterly.

| Column | Description |
|---|---|
| `ticker` | Ticker |
| `investorname` | Institution name |
| `securitytype` | `SHR`, `CLL`, `PUT`, `WNT`, `DBT`, `PRF`, `FND`, `UND` |
| `calendardate` | Quarter end date |
| `value` | Holdings value (USD thousands) |
| `units` | Number of units held |

**SF3A** — aggregate institutional ownership per ticker, quarterly. Contains total share holders, units, and values across all institutional investors, plus breakdowns by security type. Useful for computing institutional ownership percentage as a float/liquidity proxy.

**SF3B** — aggregate per investor across all holdings quarterly. Less useful for single-stock analysis.

---

## 10. SHARADAR_SFP — Fund / ETF prices

Same schema as SEP (`ticker, date, open, high, low, close, volume, closeadj, closeunadj, lastupdated`), but covering ETFs and mutual funds. Not needed for equity factor construction unless you want ETF price series for market/sector return benchmarks.

---

## 11. Fama–French daily factors

Source: [Ken French's data library](https://mba.tuck.dartmouth.edu/pages/faculty/ken.french/) (free).

Coverage: 1926-07-01 through present (updated periodically).

### File format

The file has 4 lines of plain-text header prose, then a data row with column names, then daily observations. There is a trailing copyright line at the end. Neither the header prose nor the footer are valid CSV rows — skip them on load.

```
This file was created by using the 202604 CRSP database.
The Tbill return is the simple daily rate that...
...
,Mkt-RF,SMB,HML,RF
19260701,    0.09,   -0.25,   -0.27,    0.01
19260702,    0.45,   -0.33,   -0.06,    0.01
...
Copyright 2026 Eugene F. Fama and Kenneth R. French
```

### Schema

| Column | Description | Unit |
|---|---|---|
| (unnamed) | Date in `YYYYMMDD` integer format — no column header | integer |
| `Mkt-RF` | Excess return on the market portfolio (value-weight CRSP minus RF) | percent |
| `SMB` | Small-minus-big size factor return | percent |
| `HML` | High-minus-low value factor return | percent |
| `RF` | Daily risk-free rate — simple daily rate that compounds to the 1-month T-bill over the trading month. Source: Ibbotson Associates through 2024-05; ICE BofA US 1-Month Treasury Bill Index from 2024-06 onward. | percent |

**All values are in percent — divide by 100 before using in calculations.**

### Role in this project

In practice only `RF` is used here. For Beta and Residual Volatility construction you need excess returns: subtract `RF` from each stock's daily return before running the regression on `Mkt-RF`. `Mkt-RF` itself is the regressor (market factor). `SMB` and `HML` are not needed unless you extend to a multi-factor regression.

## Factor → field mapping

Quick reference for which fields feed which USE4-style factors.

| Factor | Primary fields |
|---|---|
| **Beta** | SEP.closeadj (stock returns), Fama–French MKT-RF (market return) |
| **Size** | DAILY.marketcap or SF1.marketcap (at datekey) |
| **Non-linear Size** | Same as Size — cube of standardised Size |
| **Momentum** | SEP.closeadj (12-1 month cumulative return) |
| **Residual Volatility** | SEP.closeadj (std dev of residuals from Beta regression) |
| **Liquidity** | SEP.volume × SEP.close (DASTD / ATVR variants); SF1.revenue for size normalisation |
| **Earnings Yield** | SF1.ebit (ART) / DAILY.ev, or SF1.eps (ART) / SEP.close — **predicted E/P is absent from SF1** (no analyst consensus data); trailing earnings-based proxies only |
| **Dividend Yield** | SF1.divyield, or ACTIONS (dividend) / SEP.close |
| **Book-to-Price** | SF1.equity (ARQ) / DAILY.marketcap — **preferred equity is absent from SF1**; `equity` already excludes it per GAAP (common + APIC + retained earnings + AOCI), but the preferred equity component cannot be broken out or verified |
| **Leverage** | SF1.debt / SF1.assets; SF1.debtnc / SF1.equity; SF1.liabilities / SF1.equity — **preferred equity is absent**; cannot compute debt + preferred / total capital (the USE4 leverage definition) |
| **Growth** | SF1.revenue and SF1.eps (ARY, 5-year history) — **long-term predicted earnings growth is absent from SF1** (no analyst consensus data); historical realised growth only |
| **Non-linear Beta** | Derived from Beta |

---

## Common pitfalls

**Missing analyst consensus data** — SF1 contains only reported (historical) financials. Three USE4 inputs are entirely absent:
- *Predicted earnings-to-price ratio* (Earnings Yield descriptor): no forward EPS or analyst E/P estimates. Only trailing earnings proxies (ebit/ev, eps/price) are available.
- *Long-term predicted earnings growth* (Growth descriptor): no analyst long-term growth estimates. Only historically realised revenue and EPS growth can be computed.
- *Preferred equity* (Leverage and Book-to-Price descriptors): SF1 does not carry a preferred equity line item. `equity` is the common-equity figure (common stock + APIC + retained earnings + AOCI) and does not allow preferred equity to be added back. The USE4 leverage definition (debt + preferred / total capital) cannot be constructed precisely from SF1 alone.

**Look-ahead bias** — always filter `SF1.datekey <= signal_date`. The filing date (datekey in AR* dims) is typically 30–90 days after the fiscal period ends, so using `calendardate` or `reportperiod` as your cutoff will leak future information.

**Dimension mixing** — if you use ART for some rows and ARQ for others within the same cross-section you'll get inconsistent TTM vs quarterly numbers. Pick one dimension per factor and stick to it.

**Ticker recycling** — when a company is delisted its ticker can be reused by a new company. Sharadar handles this by appending a number to the old ticker (e.g., `FOO` vs `FOO_old`). Use `permaticker` from TICKERS as a stable entity key if you need to chain history across ticker changes.

**Split-adjusted vs total-return returns** — SEP.close is split-adjusted but not dividend-adjusted. For anything that requires total returns (momentum, beta, residual vol) use `closeadj`.

**Missing values vs genuine zero** — Sharadar imputes 0 where a field is not present on the filing. A zero `rnd` usually means no R&D, but a zero `intexp` for a highly-leveraged company often means the field is missing, not truly zero. Check the INDICATORS descriptions for which fields default to 0 vs null.

**Market cap timing** — `SF1.marketcap` is the market cap as of `datekey` (filing date). `DAILY.marketcap` gives you the market cap every calendar day. For ESTU construction or factor standardisation at a monthly rebalance date, DAILY is more appropriate.

**sharefactor** — for ADRs and dual-class structures, `sharesbas × price` is not the correct market cap; you need `sharesbas × price × sharefactor`. All the pre-computed ratio fields in SF1 already incorporate `sharefactor`; watch out only when computing from raw fields yourself.
