# Public Acceptance Criteria

This document and the [challenge README](../README.md) define the candidate contract.
Use these criteria to validate your solution. They do not prescribe an internal architecture.

## Scope And Runtime

Use the suggested **6–8 focused hours** within the one-week elapsed submission window.
Document unfinished work and priorities rather than exceeding that timebox.
Deadline and accessibility accommodations carry no penalty.

Use Python 3.12, DuckDB, and meaningful pytest tests. Support the documented
[Docker Compose build, CLI, test, and pipeline commands](../starter-kit/README.md).
Local Python setup is an optional convenience. Accept configurable `--input-dir`
and `--database` paths.

Implement at least one meaningful core transformation in SQL: current-state
selection or daily report aggregation. Table creation or simple output queries alone
do not satisfy this requirement. Either transformation is sufficient.

Full core score does **not** require:

- Cloud deployment or a cloud account.
- Workflow orchestration.
- File checkpointing or stateful file-consumption tracking.
- CI.
- Production observability.
- Historical/SCD models.
- Implemented disaster recovery for unexpected infrastructure failure or process termination.

You may describe production improvements. Optional bonuses do not replace core correctness.
Fatal-source preflight and preservation, described below, are required implemented behavior.

## Source Contract And Failure Boundary

`depots.csv` and the supplied `bookings_01.csv`, `bookings_02.csv`, and
`bookings_03.csv` are immutable. Keep them in the input directory.
Do not move, rename, edit, or delete them.

Every run may reread all source files. Process all `bookings_*.csv` files in ascending
filename order, with rows in source order. All four supplied files are required
on every run, including reruns. No file checkpoint is required.

The expected headers are:

```text
depot_id,depot_name,country
```

```text
booking_id,pickup_depot_id,dropoff_depot_id,pickup_at,dropoff_at,status,updated_at
```

Missing any supplied required file, malformed CSV structure, incorrect row width,
malformed quoting, invalid headers, or duplicate depot IDs are **fatal source-structure
errors**. A blank depot ID is also fatal.

Check the complete input for these errors **before any output mutation**.
On a fatal source-structure error:

1. Exit non-zero.
2. Add no successful-run audit rows.
3. Preserve all previously committed required models, including their schemas and data.

Do not partially refresh earlier files before discovering a structural error in a
later file. This rule requires implementation, not only a documented recovery plan.

Booking business-rule failures instead belong in `rejected_records` and do not
fail the run. Reject missing booking IDs, unknown pickup or drop-off depots,
invalid UTC timestamps, pickup at or after drop-off, and unsupported statuses.
Accept only `confirmed`, `completed`, or `cancelled` statuses.
Accept ISO 8601 UTC timestamps written with `Z` or `+00:00`, and normalize consistently.
Reject timestamps with any other offset (for example `+02:00`) or with no offset.
Preserve rejected values as their original strings.

Full disaster recovery for unexpected infrastructure failure or process termination
is not required. Document remaining partial-write risk. A transaction is strong
protection, but fatal-source preflight before output mutation is the core requirement.
Use separate temporary fixture copies for error tests. Do not change the supplied inputs.

## Required Model Contract

Create all five models in the default **`main` schema** of the configured DuckDB
database. They must remain queryable after process exit.

Every listed column is mandatory and **NOT NULL**. Extra useful columns are allowed.
For tables, enforce NOT NULL constraints. For views, every returned required value
must be non-null. Logical keys must be unique. Enforce physical primary keys for
every required model implemented as a table. Foreign-key constraints are not required.

Required views must query **persisted DuckDB data only**. After execution, they
must not read source CSV files directly or indirectly. No live `read_csv*` or other
external-file dependency is allowed in required views.

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

Keep exactly one latest valid version per booking, including currently cancelled
bookings. Validate before version selection. Invalid updates must not replace valid
state. Older valid versions and exact duplicates must not change current state.
A booking absent from a later incremental file is not deleted.

The supplied data contains no differing records with equal `booking_id` and
`updated_at`. Document a deterministic policy for that conflict.

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

Write one row per successful run and booking file. Use one shared `run_id` for the
execution. A new execution has a new run ID. No rows are allowed for a failed
execution. There is no required depot audit row.

For supplied data, classification precedence is:

1. Validate. Invalid booking rows are rejected, not duplicate, stale, or applied.
2. Check for an exact duplicate within the execution or against current state.
3. Check whether a valid non-duplicate row is older than current state: stale.
4. Otherwise, insert a new booking or a strictly newer version: applied.

Compare all seven booking values for duplicates, with valid UTC timestamps
normalized consistently. Keep earlier valid rows in the execution's duplicate
comparison even when a newer current version replaces them.

For each file and execution, the supplied fixtures satisfy:

```text
input_row_count = valid_row_count + rejected_row_count
valid_row_count = applied_row_count + duplicate_row_count + stale_row_count
```

These are per-execution counts, not counts of distinct booking IDs or net new
rejection inserts. Rejected rows still count when their logical rejection already exists.
An applied version can later be replaced during the same execution.
Equal-timestamp differing records are outside the supplied data. Document a
deterministic policy for them.

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

Build the report from current bookings whose status is `confirmed` or `completed`.
Exclude `cancelled` bookings. Count pickups by the UTC date of `pickup_at` and
`pickup_depot_id`. Count drop-offs independently by the UTC date of `dropoff_at`
and `dropoff_depot_id`.

Counts must be nonnegative. Include only rows where either count is greater than
zero. Zero-fill the other count when only one event type occurs.

## Verified Fixture Outcomes: Fresh Database, First Execution

These acceptance examples help you self-validate. They are **not permission to
hardcode fixture results**. Implement the source and model contracts.

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

```csv
booking_id,pickup_depot_id,dropoff_depot_id,pickup_at,dropoff_at,status,updated_at
B001,LIS,OPO,2026-04-11T09:00:00Z,2026-04-13T16:00:00Z,confirmed,2026-03-03T10:00:00Z
B003,FAO,LIS,2026-04-12T08:00:00Z,2026-04-15T15:00:00Z,cancelled,2026-03-04T12:00:00Z
B005,BCN,FAO,2026-04-14T07:00:00Z,2026-04-18T19:00:00Z,confirmed,2026-03-01T14:00:00Z
B006,LIS,MAD,2026-04-15T10:00:00Z,2026-04-19T17:00:00Z,cancelled,2026-03-07T09:00:00Z
```

B001's source `updated_at` uses `+00:00`. The example normalizes it to `Z`.
Compare timestamp instants, not display spelling.

## Repeated Execution

Repeat against the same completed database and unchanged input directory.
Logical current bookings, logical rejected records, and the daily report must remain
identical. Successful audit rows may append per execution.

The repeated execution's `applied_row_count` total must be **0**.
Both accounting identities must hold. No separate fixed repeat-run duplicate/stale
split is required beyond the defined classification algorithm and invariants.
Full rereads are permitted. Checkpointing is not required for incremental correctness.
