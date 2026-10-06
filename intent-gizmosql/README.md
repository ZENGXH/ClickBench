# intent-gizmosql

A fork of GizmoSQL v1.38.0 (a Flight SQL server embedding DuckDB) and of its embedded DuckDB
v1.5.5, with server and engine changes:

- GizmoSQL: https://github.com/bddppqs/intent-gizmosql, tag `v1.38.0-intent.4` (the release this entry installs: portable Linux amd64 and
  arm64 builds); changes in its `CHANGELOG.md` and `CLICKBENCH-FORK.md`.
- DuckDB: https://github.com/bddppqs/intent-duckdb, tag `v1.5.5-intent.4`; changes in its `CLICKBENCH-FORK.md`.

The scripts are the upstream `gizmosql` entry's except: `install` downloads the pinned release zip and verifies it and
both binaries by SHA-256; `query` runs the release's `gizmosql_client`, one request per query, and detects a failed
run from its exit status and error lines, not result rows; `util.sh` applies the
configuration below and polls the server's start and stop every 0.05 s instead of every second, also for the restart inside the timed `load`; `check` is described below. The schema and load path are upstream's;
the entry itself adds no index, pre-aggregation or query-specific setting.

Configuration: on machines with more than 64 CPUs `util.sh` sets one DuckDB thread per two CPUs and allocator settings
for a large dedicated host, as tuning for this benchmark and not suggested defaults; smaller machines run the defaults.
`check` runs `SELECT 1` and does not read the database file.

Caches: for its lifetime the server keeps decoded DICT_FSST dictionaries, column-wide string dictionaries, the stored
code translations and row-group index entries it has read, string-filter outcomes per dictionary entry (and the segments
they let a scan skip), and optimized plans of repeated read-only statements. None holds a query result; every run still
scans, filters and aggregates, and the restart before every cold run clears them. For a statement with a cached plan,
the hash-aggregate state is released on a background thread after its result is sent.

Within one statement, a Top-N scan's threads share one bound, which paces the row groups they take.

Storage: database files created at the latest storage version, as the entry's are, use storage version `0x40000003`,
which upstream DuckDB does not open. Blocks are zstd-compressed (level 9 with 64 or more threads, otherwise 3;
`zstd_block_compression_level` overrides it). Checkpoints also store large string columns' dictionary codes apart from
the strings, numbered column-wide, and an index of row-group positions and column statistics that the engine creates
automatically for every table and reads on a table's first scan; string statistics keep the smallest non-empty value.

The entry is `tuned: yes`: for the configuration above, for engine thresholds set on this workload and three
per-architecture build-time defaults (on for x86-64, off for arm64; all named in the fork documents), and because the
changes were developed and evaluated against the 43 ClickBench queries.

Both repositories keep their upstream licenses (DuckDB: MIT; GizmoSQL: Apache-2.0).
