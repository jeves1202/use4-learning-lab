# Usage

## Setup

```bash
uv sync                      # install polars + the numerical stack
uv run python -m ipykernel install --user --name use4-lab   # optional: named kernel
```

Put raw Sharadar CSVs where `data/README.md` says. Outputs you build go to
`data/out/` (parquet); a cleaned intermediate layer in `data/cleaned/` is worth
building once you tire of re-parsing CSVs.

## The loop, per step

1. **Read the spec notebook** (`*_spec.ipynb`) end to end. It is markdown-only:
   USE4 quotes, every NOT-IN-PDF judgment call with its default, validation
   targets, and known pitfalls.
2. **Write the build notebook** (`*_build.ipynb`) yourself — by hand or by
   driving an AI assistant with the spec as context. The spec's stage structure
   is your outline.
3. **Run it, then audit ruthlessly.** Print dataframes. Check nulls, mins,
   medians, maxes, quintiles, month-over-month stability.
4. **Compare against the reference audit** (`*_audit.md`). Reference numbers
   come from a specific data vintage — 2026-06-10 for steps 01–04; the risk-model
   audits (05–08) each state their own, slightly later, run date. Your numbers
   will differ in row counts and tails, but distribution shapes, stability
   coefficients, calibration statistics, and check outcomes should match. A
   flipped sign or a wrong order of magnitude is your bug, not vintage drift.

The loop is identical for the risk-model steps (05–08); only the checks change
character. Exposure builds validate distributions and stability; the risk-model
builds validate *calibration*: bias statistics near 1, PSD covariance forecasts,
Euler contributions that sum exactly to portfolio risk. Every gate is spelled
out in the spec's validation contract.

## Order

`00_data_cleaning` first — every downstream step reads from `data/cleaned/`.
Then `01_estu`, `02_style_factors` (with the `01.5_daily` panel extracted after
Beta and BP — see `docs/FACTOR_OVERVIEW.md` for the dependency edges),
`03_industry_factors`, and `04_country_factor`.

Steps 05–08 then build the risk model on top, strictly in order with one
exception: `05_csr` (the monthly production CSR, then the daily sibling) →
`06_fcov` → `07_specific_risk` → `08_risk_decomp`. The exception: 07 consumes
the **daily CSR residuals from 05 directly** — it does not depend on 06, so you
can build 07 before or in parallel with 06. Step 08 composes everything and
runs last. Note that `05_csr` ships two specs (monthly and daily) but one audit;
the daily build's reference audit lands with the next audit refresh.

## Conventions worth copying

- **Point-in-time or it didn't happen**: `datekey ≤ signal_date`, never
  `calendardate`. In the risk model the same discipline shows up as: exposures
  at t explain only returns strictly after t, and every forecast at t uses data
  dated ≤ t.
- **Standardize on ESTU, apply to everyone**: cap-weighted mean 0,
  equal-weighted std 1.
- **Validation batteries are non-negotiable**: every build ends with explicit
  PASS/FAIL checks, and coverage checks exclude the in-progress final month.
- **When a stat is out of band and you understand why — flag it, don't tune it
  away.** Our audits carry WARN entries on purpose.
