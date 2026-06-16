# Estimation Universe (ESTU) — Construction Summary

**MSCI USA IMI Proxy — `estu_build.ipynb`**
Run Date: 2026-06-11 | USE4 Learning Lab — Reference Audit

---

## Purpose and Scope

Summarises the fresh build of the shared estimation universe `data/out/estu_monthly.parquet` — the MSCI USA IMI proxy consumed by every factor build. Construction: domestic common stocks on NYSE/NASDAQ/NYSEMKT, $300M mcap floor, ATVR liquidity screen (add ≥20%, retain ≥10%, 63-day window, min 42 samples), top-3,000 cap target with a 3,000/3,500 hysteresis buffer. All values are verbatim from the run dated 2026-06-11.

> *Reference numbers: every value in this document comes verbatim from the lab's reference run of 2026-06-11 (Sharadar data through 2026-06-10). Your rebuild will differ in row counts and tail statistics with data vintage; distribution shapes, stability coefficients, and check outcomes should match. Treat persistent sign or order-of-magnitude differences as bugs in your build until proven otherwise.*

---

## Build Diagnostics

- Prices: **39,015,418** rows, **17,419** permatickers; **7,154** trading days (1997-12-31 → 2026-06-10); FAST mode
- Path-invariance check: max diff **0.00×10⁰** ✅ PASS
- Signal calendar: **342 dates** (1998-01-30 → 2026-06-10)
- Snapshot panel: **1,863,775** rows
- Latest date (2026-06-10) funnel: 4,706 total → 2,842 hard filter → 2,715 / 2,738 liq-add / liq-retain → segments large 300, mid 399, small 2,025
- ESTU size: mean **2,492**, min **1,791**, max **3,045**; churn mean **2.3%** (worst month 2009-04: 157 in / 18 out)
- Deliverable: **1,863,775** rows, **17,388** tickers, in-ESTU **825,015** rows, **20.0 MB**; 12-column schema check ✅ PASSED; read-back ✅ PASSED

---

## Known Deviations from USE4 / MSCI IMI

Mirrors the complete NOT-IN-PDF master list from `estu_spec.ipynb` (§14). Rejected alternatives are documented in the guide.

| # | Decision | Chosen solution | Revisit when |
|---|---|---|---|
| 1 | Float-adjusted vs total mcap ⚠️ **Material** | Total mcap (Sharadar lacks float) | If validating against MSCI IMI directly |
| 2 | Allowed exchanges | NYSE, NASDAQ, NYSEMKT | If we want broader coverage |
| 3 | Allowed categories | Domestic common stock + REITs | If REIT exposures distort risk model |
| 4 | ADRs included? | No (foreign issuer) | If portfolio holds ADRs that need risk forecast |
| 5 | Liquidity metric | ATVR proxy (63d median dollar vol × 252 / mcap): add ≥ 20%, retain ≥ 10% | If small-cap segment looks noisy |
| 6 | Liquidity window length | 63 trading days | If liquidity oscillates dramatically |
| 7 | Liquidity history requirement | ≥ 42 of 63 days with data | Affects how quickly IPOs enter ESTU |
| 8 | Cap target mechanism | Top 3000 by mcap | If we want cleaner match to USE4 spec literal text |
| 9 | Buffer add threshold | rank ≤ 3000 | If churn rate is wrong direction |
| 10 | Buffer drop threshold | rank > 3500 | If too many borderline names flip in/out |
| 11 | Rebalance frequency | Monthly | If month-to-month is noisy or expensive |
| 12 | Calendar start | 1998-01-01 | Earliest date Sharadar has reliable mcap data |
| 13 | Price floor | None (rely on liquidity) | If liquidity screen fails to catch known cases |
| 14 | Sector caps | None | If sector tilts in ESTU distort factor returns |
| 15 | Special handling for new IPOs | 42-day minimum history (implicit) | If post-IPO names destabilize ESTU |

---

## Validation Checks

All 8 checks passed on the run dated 2026-06-11.

| Check | Description | Observed | Target / Status |
|---|---|---|---|
| 1 | ESTU size per date | recent 2,187–3,045; early 1,791–2,760 | 2,100–3,200 / 1,500–3,200 ✅ PASS |
| 2 | Month-over-month churn | mean 2.3%; p95 4.8%; 95.5% < 5% | ≥95% months <5% ✅ PASS |
| 3 | Mega-cap inclusion | AAPL 257mo; GOOGL 261mo; MSFT 331mo; JPM 331mo | whenever eligible ✅ PASS |
| 4 | Mcap coverage (ESTU/retain-elig) | mean 99.3%; min post-2005 95.8% | ≤100%, min ≥93% ✅ PASS |
| 5 | Liquidity sanity (post-2015) | median $18M/day; p10 $1.8M/day | ≥$15M; ≥$1.5M ✅ PASS |
| 6 | Segment proportions (post-2010) | large 11.5%; mid 15.3%; small 73.2% | 8–15% / 10–20% ✅ PASS |
| 7 | No PIT leaks | 0 rows with null/zero mcap in ESTU | exactly 0 ✅ PASS |
| 8 | Buffer flag invariants | all 4 hold; bootstrap 0 | all invariants ✅ PASS |

**Check 4 note:** ESTU / hard-filtered coverage is 95.5%; the 4.5% gap is hard-filter passers with ATVR < 10% (illiquid, correctly excluded).

---

## Spot Check — ESTU mcap floor over time

The smallest in-ESTU market caps confirm the $300M floor binds exactly and drifts upward only through the p5/p10 quantiles:

| Date | min | p5 | p10 |
|---|---|---|---|
| 2000-12-29 | $300.0M | $347.7M | $392.8M |
| 2005-12-30 | $300.2M | $346.0M | $399.8M |
| 2010-12-31 | $300.3M | $358.3M | $423.6M |
| 2015-12-31 | $300.7M | $368.4M | $440.8M |
| 2020-12-31 | $300.0M | $375.1M | $464.5M |
| 2026-06-10 | $300.7M | $380.3M | $481.4M |

All-time overall min **$300.0M**; mean of per-date mins **$300.4M**.
