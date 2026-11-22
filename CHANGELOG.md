# Changelog

All notable changes to this package. Versions follow semver, independent of the Nabi toolchain.

## 0.1.1 (2026-11-22)

- The package docs open with the data analysis tutorial (`table`,
  `stats`, `plot`), moved here from the Nabi docs site; each step is a
  doc test.

## 0.1.0 (2026-11-21)

First release.

- Typed columns (`i64`, `f64`, `bool`, `str`, `cat`, `date`, `datetime`)
  with nulls.
- I/O: CSV, JSON lines, Arrow IPC and Parquet, without a C library.
- Column expressions with the dot operators, sorting, group-by, hash joins,
  pivot / melt, window functions and summaries.
- Lazy queries (`lazy`, `scan_parquet`, `scan_arrow`, `scan_csv`) with
  filter and projection pushdown.
- `table::ops` relational operators (joins and set operations).
- `date"..."` / `datetime"..."` literals, `table!`, `column!` and
  `#[derive(table::Record)]`.
- Needs Nabi 0.1 or later; depends on `csv` and `json`.
