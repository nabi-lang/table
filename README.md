# table

[![CI](https://github.com/nabi-lang/table/actions/workflows/ci.yml/badge.svg)](https://github.com/nabi-lang/table/actions/workflows/ci.yml) [![packages.nabi-lang.org](https://img.shields.io/github/v/tag/nabi-lang/table?label=packages.nabi-lang.org&sort=semver)](https://packages.nabi-lang.org/packages/table) [![License: MIT OR Apache-2.0](https://img.shields.io/badge/license-MIT%20OR%20Apache--2.0-blue)](#license)

Column-oriented tables for Nabi: typed columns (`i64`, `f64`, `bool`,
`str`, dictionary-encoded `cat`, `date`, `datetime`) with nulls, CSV
([RFC 4180](https://www.rfc-editor.org/rfc/rfc4180)) and JSON lines I/O,
Arrow IPC and Parquet I/O, column expressions written with the dot
operators, sorting, group-by, hash joins, pivot / melt, window functions,
summaries and lazy queries with filter and projection pushdown. Grouping and
joins hash rows into integer keys, so nothing is allocated per row; large
group-bys and sorts use every core.

## Add it

```sh
nabi add table
```

Needs Nabi 0.1 or later.

## Usage

```nabi
use table::{col, desc, mean, sum, when};

let t = table::table!{
    "city": ["Seoul", "Busan", "Seoul", "Daegu", "Busan"],
    "day":  [date"2024-07-01", date"2024-07-01", date"2024-07-02", date"2024-07-02", date"2024-07-02"],
    "temp": [27.5, 29.0, none, 31.5, 28.0],
    "rain": [0.0, 12.5, 3.0, 0.0, 1.0],
};

// Expressions: `.+ .- .* ./ .**`, comparisons `.> .== ..`, `.&& .||`.
let t = try t.with_column("feels", col("temp") .+ col("rain") .* 0.1);
let hot = try t.filter(col("temp") .>= 28 .&& col("city") .!= "Daegu");

// Group, aggregate, sort.
let by_city = try (try t.group_by(["city"])).agg([mean("temp"), sum("rain").named("total_rain")]);
println(try by_city.sort([desc("total_rain")]));

// Conditionals and windows.
let t = try t.with_column("wet", when(col("rain") .> 2).then("yes").otherwise("no"));
let t = try t.with_column("rain_3", col("rain").rolling_mean(3));
```

```text
shape: (3, 3)
city  | temp_mean | total_rain
str   | f64       | f64
------+-----------+-----------
Busan |      28.5 |       13.5
Seoul |      27.5 |        3.0
Daegu |      31.5 |        0.0
```

Read and write files with `read_csv` / `write_csv` and `read_jsonl` /
`write_jsonl`, which read well with the pipe; `Csv` sets the delimiter,
null tokens and column types, and `Csv::reader` streams a large file in
batches. CSV goes through the [`csv`](https://packages.nabi-lang.org/packages/csv)
package; `Csv::dialect` gives its `Dialect` for these options:

```nabi norun
let t = try "weather.csv" |> table::read_csv;
try t |> table::write_jsonl("weather.jsonl");

let opts = table::Csv::new().delimiter(';').types([("zip", table::DType::Str)]);
let a = try opts.read_file("addresses.csv");
```

### Arrow and Parquet

`read_parquet` / `write_parquet` and `read_arrow` / `write_arrow` read and
write the columnar formats other tools use, without a C library;
`Parquet` and `Arrow` set the options, and `columns` reads only the
columns named:

```nabi norun
let t = try "trips.parquet" |> table::read_parquet;
try t |> table::write_arrow("trips.arrow");

let p = table::Parquet::new().row_group_rows(100000).compression(table::Compression::None);
try p.write_file(t, "plain.parquet");
let fares = try table::Parquet::new().columns(["fare", "tip"]).read_file("trips.parquet");
```

Each type maps one to one: `i64` to int64 / INT64, `f64` to float64 /
DOUBLE, `bool`, `str` to utf8 / STRING, `cat` to a utf8 dictionary (a
dictionary page in Parquet), `date` to date32 / DATE, `datetime` to
timestamp[ns] / TIMESTAMP(NANOS); nulls stay nulls both ways.

| | Arrow IPC | Parquet |
|---|---|---|
| Write | file and stream format, record batches of a set size, uncompressed | data pages v1, PLAIN and dictionary encodings, row groups and pages of a set size, min / max statistics, Snappy or none, the Arrow schema in the metadata |
| Read | file and stream format, delta and replacement dictionaries, ZSTD and LZ4 frame bodies | data pages v1 and v2; PLAIN, dictionary, RLE, DELTA_BINARY_PACKED, DELTA_LENGTH_BYTE_ARRAY, DELTA_BYTE_ARRAY, BYTE_STREAM_SPLIT; Snappy, Zstandard or none |
| Read types | int8 to int64 and uint8 to uint64 (to `i64`), float16 / float32 / float64, bool, utf8, large_utf8, utf8 dictionaries, date32 / date64, timestamps of any unit (a time zone is dropped, values stay UTC), null | INT32 / INT64 of any integer logical type, INT96, every timestamp unit, FLOAT, DOUBLE, FLOAT16, strings |
| Not read | binary, decimal, time, duration, interval, views, nested types | nested, decimal, time and fixed-length binary columns; gzip, Brotli and LZ4 pages |

A column of a type not read is an error naming it, unless `columns`
leaves it out. Writing Zstandard is not supported (reading is): the
compressor is a large piece of work for a small gain over Snappy here.

### Lazy queries

`t.lazy()`, `scan_parquet`, `scan_arrow` and `scan_csv` start a `Query`:
the same steps as on a table, run on `collect` after an optimizer pushes
filters down (into the scan, where a Parquet row group whose statistics
rule a filter out is never read), prunes every column nothing uses (the
scan reads only the rest) and stops a scan at a limit. `explain` shows the
plan it runs:

```nabi norun
use table::{col, desc, mean};

let q = table::scan_parquet("trips.parquet")
    .filter(col("fare") .> 50.0)
    .group_by(["vendor"]).agg([mean("tip")])
    .sort([desc("tip_mean")]);
println(try q.explain());
let top = try q.collect();
```

```text
SORT [tip_mean desc]
  GROUP BY [vendor] AGG [tip_mean]
    SCAN parquet "trips.parquet" read [vendor, fare, tip] filter (col("fare") .> 50) keep [vendor, tip] row groups by [fare > 50]
```

Filters stay above a `limit`, a window function (`cum_sum`, `rolling_*`,
`shift`) and the null-filling side of an outer join, so `collect` always
gives what `collect_unoptimized` gives.

### Speed

`nabi bench formats` on one M-series core (`-O2`), a 1e6-row table of an
`i64`, an `f64` with nulls, a 37-value `str` and a `date`:

| Operation | Time |
|---|---|
| Arrow IPC write / read | 87 ms / 60 ms |
| Parquet write, Snappy / none | 153 ms / 148 ms |
| Parquet read, Snappy | 91 ms |
| Filter `id < 100000`, group-by mean: eager read then query | 98 ms |
| The same as a lazy scan (row groups of 65536 rows) | 20 ms |

Numeric columns and `core::array` arrays share one buffer:
`Column::from_array(a)` keeps a contiguous array's elements without a
copy, and `t.f64("temp")` / `c.to_f64()` hand back the column's own
buffer when it has no nulls, ready for the dot operators. Treat that
array as read-only (`.copy()` it to write), since the column may be in
other tables too.

### Relational operators

`table::ops` writes joins and set operations in relational algebra:

| Operator | Meaning                                    |
|----------|--------------------------------------------|
| `a ⋈ b`  | inner join on the shared column names       |
| `a ⟕ b`, `a ⟖ b`, `a ⟗ b` | left, right, full outer join |
| `a ⋉ b`, `a ▷ b` | semi-join, anti-join                |
| `a ⋈ b.on(["id"])` | join on named keys                |
| `a ∪ b`, `a ∩ b`, `a ∖ b` | distinct rows of both, of both, of `a` only |

```nabi
let people = table::table!{ "id": [1, 2, 3], "name": ["ann", "bob", "cy"] };
let orders = table::table!{ "id": [1, 1, 3], "amount": [5.0, 7.0, 2.0] };
let joined = people ⋈ orders;      // 3 rows
let idle = people ▷ orders;        // bob
```

The joins bind like `*` and the set operators like `+` / `-`. An operator
panics where `Table::join` would return an error, so use the method when
column names come from data.

### Literals

`date"2024-01-15"` and `datetime"2024-01-15T10:30:00"` are checked when the
program compiles; `table!{ .. }` and `column![..]` build tables and columns
from cells (`none` for a null), and `#[derive(table::Record)]` turns a list
of structs into a table with `record_table`.

## Docs

https://docs.nabi-lang.org/packages/table/latest

## License

Licensed under either of [MIT](LICENSE-MIT) or [Apache-2.0](LICENSE-APACHE), at
your option.
