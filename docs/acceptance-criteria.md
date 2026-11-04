# Public Acceptance Criteria

This document, together with the [challenge README](../README.md), defines the
candidate contract. Use it to validate your solution: it describes what the
result must satisfy, not how to structure your code internally.

## Scope And Runtime

Aim for the suggested **6–8 focused hours** within the one-week submission
window. If you run out of time, document what you left unfinished and why rather
than pushing past the timebox. Deadline and accessibility accommodations carry
no penalty.

Build on Python 3.12, DuckDB, and meaningful pytest tests, and support the
documented [Docker Compose build, CLI, test, and pipeline commands](../starter-kit/README.md).
A local Python setup is an optional convenience. Your CLI must accept
configurable `--input-dir` and `--database` paths.

At least one core transformation must be real SQL — either current-state
selection or daily report aggregation. Either one is enough on its own, but
table creation or simple pass-through output queries do not count.

A full core score does **not** require any of the following:

- Cloud deployment or a cloud account.
- Workflow orchestration.
- File checkpointing or stateful file-consumption tracking.
- CI.
- Production observability.
- Historical/SCD models.
- Implemented disaster recovery for unexpected infrastructure failure or process termination.

You are welcome to describe production improvements in writing, but optional
bonuses never substitute for core correctness. Two behaviors are the exception
and must be actually implemented, not just planned: checking the whole input for
structural errors before any output mutation (**fatal-source preflight**), and
keeping previously committed models intact when a run fails (**model
preservation**). Both are specified below.

## Source Contract And Failure Boundary

The inputs are fixed. `depots.csv` and the supplied `bookings_01.csv`,
`bookings_02.csv`, and `bookings_03.csv` are immutable — keep them in the input
directory and never move, rename, edit, or delete them.

Treat every run as a full reread. Process all `bookings_*.csv` files in ascending
filename order, and the rows within each file in source order. All four supplied
files must be present on every run, including reruns; no file checkpoint is
expected.

The expected headers are:

```text
depot_id,depot_name,country
```

```text
booking_id,pickup_depot_id,dropoff_depot_id,pickup_at,dropoff_at,status,updated_at
```

Some problems describe a broken input rather than a bad record, and these are
**fatal source-structure errors**: a missing required file, malformed CSV
structure, incorrect row width, malformed quoting, invalid headers, a duplicate
depot ID, or a blank depot ID.

Because of that, check the entire input for structural errors **before mutating
any output**. When you find one:

1. Exit non-zero.
2. Write no successful-run audit rows.
3. Preserve every previously committed required model, schema and data intact.

In particular, do not refresh earlier files and only then discover a structural
error in a later file. This is behavior you must implement, not merely a recovery
plan you describe.

Booking business-rule failures are different: they belong in `rejected_records`
and never fail the run. Reject a row for a missing booking ID, an unknown pickup
or drop-off depot, an invalid UTC timestamp, a pickup at or after its drop-off,
or an unsupported status — only `confirmed`, `completed`, and `cancelled` are
supported. Accept ISO 8601 UTC timestamps written with `Z` or `+00:00` and
normalize them consistently, but reject any other offset (for example `+02:00`)
or a missing offset. Always preserve rejected values as their original strings.

Full disaster recovery for unexpected infrastructure failure or process
termination is out of scope — just document whatever partial-write risk remains.
A transaction is strong protection, but the core requirement is the fatal-source
preflight before any output mutation. For error tests, work on separate temporary
fixture copies so the supplied inputs stay untouched.

## Required Model Contract

Create all five models in the configured DuckDB database's default **`main`
schema**, and make sure they remain queryable after the process exits.

Every listed column is mandatory and **NOT NULL**, though extra useful columns
are fine. Enforce NOT NULL constraints on tables, and for views ensure every
returned required value is non-null. Logical keys must be unique, and every
required model implemented as a table must enforce a physical primary key.
Foreign-key constraints are not required.

Required views must read **persisted DuckDB data only**. Once execution finishes
they must not touch source CSV files, directly or indirectly — no live
`read_csv*` or any other external-file dependency is allowed in a required view.

### `depots`

**Persisted table.** Logical and physical primary key: `depot_id`.

| Required column | DuckDB type | Nullability |
|---|---|---|
| `depot_id` | `VARCHAR` | NOT NULL |
| `depot_name` | `VARCHAR` | NOT NULL |
| `country` | `VARCHAR` | NOT NULL |

### `current_bookings`

**Persisted table or view over persisted DuckDB data.** Logical key: `booking_id`.
If implemented as a table, enforce `PRIMARY KEY (booking_id)`.

| Required column | DuckDB type | Nullability |
|---|---|---|
| `booking_id` | `VARCHAR` | NOT NULL |
| `pickup_depot_id` | `VARCHAR` | NOT NULL |
| `dropoff_depot_id` | `VARCHAR` | NOT NULL |
| `pickup_at` | `TIMESTAMPTZ` | NOT NULL |
| `dropoff_at` | `TIMESTAMPTZ` | NOT NULL |
| `status` | `VARCHAR` | NOT NULL |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL |

Keep exactly one latest valid version of each booking, cancelled ones included.
Validation happens before version selection, so an invalid update can never
replace valid state, and neither older valid versions nor exact duplicates change
the current state. A booking that is simply absent from a later incremental file
is not deleted.

The supplied data never contains two differing records that share a `booking_id`
and `updated_at`; even so, document the deterministic policy you would apply to
that conflict.

### `rejected_records`

**Persisted table.** Logical and physical primary key: `rejection_id`.

| Required column | DuckDB type | Nullability |
|---|---|---|
| `rejection_id` | `VARCHAR` | NOT NULL |
| `source_filename` | `VARCHAR` | NOT NULL |
| `source_row_number` | `BIGINT` | NOT NULL |
| `raw_source_json` | `JSON` | NOT NULL |
| `rejection_reasons_json` | `JSON` | NOT NULL |

- `source_filename` is the booking filename without its directory.
- `source_row_number` is the 1-based data-row number, excluding the CSV header.
- `rejection_id` must be `<source_filename>:<1-based data-row number>`.
- `raw_source_json` is an object containing all seven original booking columns as lossless strings, including empty or invalid values.
- `rejection_reasons_json` is a non-empty array of understandable strings.
- Exact reason wording is not prescribed. Document your policy for rows with multiple failures.
- Reruns must not duplicate logical rejections.

### `processing_audit`

**Persisted table.** Logical and physical primary key: `(run_id, source_filename)`.

| Required column | DuckDB type | Nullability | Meaning |
|---|---|---|---|
| `run_id` | `VARCHAR` | NOT NULL | One execution identifier shared by all booking files in the execution. |
| `source_filename` | `VARCHAR` | NOT NULL | Booking filename without its directory. |
| `input_row_count` | `BIGINT` | NOT NULL | Physical booking data rows read, excluding the header. |
| `valid_row_count` | `BIGINT` | NOT NULL | Rows passing all booking business validation. |
| `rejected_row_count` | `BIGINT` | NOT NULL | Rows failing business validation and written once logically to `rejected_records`. |
| `applied_row_count` | `BIGINT` | NOT NULL | Valid rows that insert a new current booking or a strictly newer current version during that execution. |
| `duplicate_row_count` | `BIGINT` | NOT NULL | Valid rows exactly matching a valid row encountered earlier in the execution, or exactly matching current state. No state change. |
| `stale_row_count` | `BIGINT` | NOT NULL | Valid non-duplicate rows older than current state. No state change. |
| `run_started_at` | `TIMESTAMPTZ` | NOT NULL | UTC execution start timestamp shared by all files in the execution. |

Write exactly one row per booking file per successful run, all sharing a single
`run_id`, and give each new execution a new run ID. A failed execution writes no
rows at all, and there is no depot audit row.

For the supplied data, classify each valid row in this order:

1. Validate first. Invalid rows are rejected — never duplicate, stale, or applied.
2. Check for an exact duplicate within the execution or against current state.
3. If a valid non-duplicate row is older than current state, it is stale.
4. Otherwise insert a new booking or a strictly newer version: applied.

Duplicate comparison uses all seven booking values with valid UTC timestamps
normalized consistently, and an earlier valid row stays in that comparison for
the rest of the execution even after a newer current version replaces it.

For each file and execution, the supplied fixtures satisfy:

```text
input_row_count = valid_row_count + rejected_row_count
valid_row_count = applied_row_count + duplicate_row_count + stale_row_count
```

These are per-execution counts, not counts of distinct booking IDs or of net new
rejection inserts. A rejected row still counts even when its logical rejection
already exists, an applied version can be replaced again later in the same
execution, and equal-timestamp differing records fall outside the supplied data
(document a deterministic policy for them).

### `daily_operations_report`

**Persisted table or view over persisted DuckDB data.**
Logical key: `(operation_date, depot_id)`.
If implemented as a table, enforce `PRIMARY KEY (operation_date, depot_id)`.

| Required column | DuckDB type | Nullability |
|---|---|---|
| `operation_date` | `DATE` | NOT NULL |
| `depot_id` | `VARCHAR` | NOT NULL |
| `pickup_count` | `BIGINT` | NOT NULL |
| `dropoff_count` | `BIGINT` | NOT NULL |

Build the report from current bookings whose status is `confirmed` or
`completed`, excluding `cancelled` ones. Count pickups by the UTC date of
`pickup_at` together with `pickup_depot_id`, and count drop-offs independently by
the UTC date of `dropoff_at` together with `dropoff_depot_id`. All counts are
nonnegative. Include a row only when at least one of its counts is greater than
zero, and zero-fill the other count when only one event type occurs.

## Verified Fixture Outcomes: Fresh Database, First Execution

These acceptance examples let you self-validate. They are **not permission to
hardcode fixture results** — implement the source and model contracts and let
these numbers fall out.

| Audit measure | Total |
|---|---:|
| `input_row_count` | 28 |
| `valid_row_count` | 19 |
| `rejected_row_count` | 9 |
| `applied_row_count` | 15 |
| `duplicate_row_count` | 2 |
| `stale_row_count` | 2 |

| Final model measure | Expected |
|---|---:|
| Depots | 5 |
| Current bookings | 9 |
| Cancelled current bookings | 2 |
| Report-eligible current bookings (`confirmed` or `completed`) | 7 |
| Daily report rows | 11 |
| Total pickups | 7 |
| Total drop-offs | 7 |

The following sentinel current versions show **all seven required booking fields**:

| `booking_id` | `pickup_depot_id` | `dropoff_depot_id` | `pickup_at` | `dropoff_at` | `status` | `updated_at` |
|---|---|---|---|---|---|---|
| B001 | LIS | OPO | 2026-04-11T09:00:00Z | 2026-04-13T16:00:00Z | confirmed | 2026-03-03T10:00:00Z |
| B003 | FAO | LIS | 2026-04-12T08:00:00Z | 2026-04-15T15:00:00Z | cancelled | 2026-03-04T12:00:00Z |
| B005 | BCN | FAO | 2026-04-14T07:00:00Z | 2026-04-18T19:00:00Z | confirmed | 2026-03-01T14:00:00Z |
| B006 | LIS | MAD | 2026-04-15T10:00:00Z | 2026-04-19T17:00:00Z | cancelled | 2026-03-07T09:00:00Z |

B001's source `updated_at` uses `+00:00`, and the example normalizes it to `Z`.
Compare timestamp instants, not their display spelling.

## Repeated Execution

Run the pipeline again against the same completed database and unchanged input
directory. The logical current bookings, logical rejected records, and daily
report must come out identical; only successful audit rows may append.

On that repeat, the total `applied_row_count` must be **0** and both accounting
identities must still hold. Beyond the classification algorithm and invariants
already defined, no separate duplicate/stale split is prescribed for repeat runs.
Full rereads are fine, and checkpointing is not required for incremental
correctness.
