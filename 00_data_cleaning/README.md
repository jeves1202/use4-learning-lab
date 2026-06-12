# Step 0 — Sharadar Cleaning (planned)

**Status: not yet built.** This step will be built by Maxwell; it will be added when the reference implementation exists.

## What this step will cover

Every downstream step reads from `data/cleaned/` rather than the raw CSVs in
`data/raw/`. The cleaning pipeline is the one-time translation layer between
what Sharadar ships and what the rest of the repo expects:

1. **Parse and type raw CSVs.** Sharadar delivers everything as flat CSVs.
   Each table needs dates parsed, numeric columns cast, and known sentinel
   values (e.g. empty strings standing in for nulls) resolved.
2. **Partition and compress.** Large files (SF1 ~2.4 GB, SEP ~3.2 GB) are
   partitioned by year and written to zstd-compressed parquet. This turns
   multi-second CSV scans into sub-second parquet reads for every downstream
   notebook.
3. **Unit normalisation.** Sharadar `marketcap` is in millions of dollars;
   downstream code expects dollars. Currency conversions, share-factor
   adjustments, and any other field-level normalisation decisions are fixed
   here once so they never need to be re-litigated in factor notebooks.
4. **Split-factor history.** Cumulative split factors derived from ACTIONS are
   joined into the prices table so that downstream consumers get clean,
   split-consistent series without touching ACTIONS themselves.
5. **Deduplication.** Dual-class share-class rows and any vendor duplicates are
   resolved to one canonical row per `(permaticker, date)`.

## Prerequisite

Raw Sharadar CSVs placed at the paths documented in `data/README.md`. No other
steps are required — this is step zero.
