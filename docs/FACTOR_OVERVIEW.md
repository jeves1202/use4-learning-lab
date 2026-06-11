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
