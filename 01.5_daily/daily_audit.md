# Daily Returns Artifact — Construction Summary

**Shared Time-Series Infrastructure — `daily_build.ipynb`**
Run Date: 2026-06-12 | USE4 Learning Lab — Reference Audit

---

## Purpose and Scope

Summarises the fresh build of the shared daily panel consumed by the five time-series factor builds (`beta`, `mom`, `nlb`, `nls`, `resvol`): RF-adjusted excess log returns with the ESTU cap-weighted excess benchmark, built once so the factor notebooks start from a fast, bounded parquet read. All values are verbatim from the run dated 2026-06-12.

> *Reference numbers: every value in this document comes verbatim from the lab's reference run of 2026-06-12 (Sharadar data through 2026-06-10). Your rebuild will differ in row counts and tail statistics with data vintage; distribution shapes, stability coefficients, and check outcomes should match. Treat persistent sign or order-of-magnitude differences as bugs in your build until proven otherwise.*

---

## Build Diagnostics

- Daily RF (Ken French): **26,233** days (1926-07-01 → 2026-04-30); mean **3.01%** annualised
- Prices: **39,015,418** rows, **17,419** tickers; mode **FAST** (51.6 GB RAM avail, commit headroom 84.5 GB)
- Returns after null drop: **38,997,999** rows, **17,413** tickers; market calendar **6,922** trading days (1998-12-02 → 2026-06-10)
- FAST ≡ SEQUENTIAL equivalence: ✅ **PASS** (<10⁻¹² on returns and lagged mcap)
- ESTU benchmark: **6,902** trading days; correlation vs broad CRSP proxy **0.9334** (target >0.90); full-sample ann. vol **19.5%**; GFC peak **55.5%**; COVID peak **69.0%** (targets >40%); missing-benchmark dates **0** (target <30)

---

## Known Deviations from USE4

Mirrors the complete NOT-IN-PDF master list from `daily_spec.ipynb` (§10) — every judgment call the USE4 PDFs do not specify, with the chosen solution and its revisit trigger.

| # | Decision | Chosen solution | Revisit when |
|---|---|---|---|
| 1 | RF source | Ken French daily RF | Validating against USE4 published numbers |
| 2 | Dual-class dedup | Keep highest-mcap row per `(permaticker, date)` | Share-class-level modelling is needed |
| 3 | Benchmark weights | `mcap_lag` (prior day, total mcap — Sharadar has no float) | Float data is procured |
| 4 | Memory strategy | Commit-aware FAST/SEQ gate (≥10× RAM and ≥20× commit headroom) | Hardware changes significantly |
| 5 | Membership mapping | Owning signal date = most recent month-end ≤ trading date | Intramonth membership changes are shown to matter |

---

## Artifacts Written

All written to `data/out/`, zstd compressed, read-back asserted.

| File | Rows | Size | Description |
|---|---|---|---|
| `daily_returns.parquet` | 38,997,999 | 377.9 MB | Primary artifact — one row per (permaticker, date) |
| `estu_mkt.parquet` | 6,902 | 0.1 MB | ESTU cap-weighted excess benchmark series |
| `crsp_mkt.parquet` | 6,922 | 0.1 MB | Broad CRSP-proxy benchmark (diagnostics only) |
| `ticker_permaticker.parquet` | 17,456 | 0.1 MB | Ticker → permaticker symbol map |

Schema of `daily_returns.parquet`:

| Column | Dtype | Description |
|---|---|---|
| `permaticker` | Int64 | Sharadar permanent ticker ID |
| `date` | Date | Trading day |
| `ret` | Float64 | Excess log return: `ln(closeadj_t / closeadj_{t-1}) − rf_t` |
| `mcap_lag` | Float64 | Prior trading day's market cap (benchmark weight — known before the return accrues) |
| `mkt_ret` | Float64 | ESTU cap-weighted excess benchmark return on `date` |

**Freshness contract:** `daily_returns.parquet` must cover data through the last ESTU signal date. Downstream factor builds should assert this before loading — if the artifact is stale, rolling windows silently truncate.
