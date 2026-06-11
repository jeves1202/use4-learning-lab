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
4. **Compare against the reference audit** (`*_audit.tex`). Our numbers come
   from a specific data vintage (2026-06-10) — yours will differ in row counts
   and tails, but distribution shapes, stability coefficients, and check
   outcomes should match. A flipped sign or a wrong order of magnitude is your
   bug, not vintage drift.

## Order

`01_estu` first, always. Then `02_style_factors` (see
`docs/FACTOR_OVERVIEW.md` for a suggested path and the dependency edges).
Steps 03–04 are planned; the repo updates when they land.

## Conventions worth copying

- **Point-in-time or it didn't happen**: `datekey ≤ signal_date`, never
  `calendardate`.
- **Standardize on ESTU, apply to everyone**: cap-weighted mean 0,
  equal-weighted std 1.
- **Validation batteries are non-negotiable**: every build ends with explicit
  PASS/FAIL checks, and coverage checks exclude the in-progress final month.
- **When a stat is out of band and you understand why — flag it, don't tune it
  away.** Our audits carry WARN entries on purpose.
