# Step 3 — Industry Factors (planned)

**Status: not yet built.** This step lands after the style factors; the repo will
update when the reference implementation exists. Nothing here is final.

## What this step will cover

USE4 assigns every stock to one of ~60 GICS industries and gives it unit
exposure to that industry factor (industries are dummies, not standardized
descriptors). The work, in practice:

1. **Classification mapping.** Sharadar carries SIC codes, `sicsector` /
   `sicindustry`, a Fama–French classification, and its own `sector` /
   `industry` fields — but not GICS. Building a defensible GICS-like scheme
   from these (and keeping it point-in-time: companies get reclassified) is
   the real data problem of this step.
2. **Exposure construction.** One-hot membership per (stock, signal date),
   with rules for conglomerates, reclassifications, and names with stale
   metadata.
3. **Audit.** Industry counts over time, membership churn, coverage against
   the ESTU, and sanity against known sector events.

## Prerequisite

Steps 1–2 complete. The industry dummies only matter inside the cross-sectional
regression alongside your style exposures — without the styles there is nothing
to regress.
