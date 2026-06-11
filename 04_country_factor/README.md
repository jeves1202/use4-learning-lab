# Step 4 — Country Factor (planned)

**Status: not yet built.** This is the final exposures step and the bridge to
the factor-returns / covariance repo; it will be added when the reference
implementation exists.

## What this step will cover

In a single-country (US) model the country factor's *exposure* is trivial —
every stock gets exposure 1. The substance is everything around it:

1. **Why it exists.** The country factor absorbs the market's common return so
   industry and style factors price *relative* effects. Without it, your
   industry factors each smuggle in a copy of the market.
2. **The constraint.** Country + cap-weighted industries are perfectly
   collinear; USE4 resolves this with a constraint on industry factor returns
   in the cross-sectional regression. Understanding this is the whole step.
3. **The bridge.** Once you have country + industry + style exposures, the
   next repo runs the constrained weekly/monthly cross-sectional regressions
   that turn exposures into factor returns — and from there, the covariance
   matrix.

## Prerequisite

Steps 1–3 complete, and a working read of USE4 Methodology Notes on the
regression structure (constraints, $\sqrt{\text{mcap}}$ weighting).
