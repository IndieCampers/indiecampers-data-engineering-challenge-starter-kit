# Booking Operations Data Pipeline

Welcome to the Indie Campers data engineering challenge.

This exercise is designed for an entry-level or junior data engineer. It focuses on practical data engineering fundamentals: reading incremental files, validating data, maintaining current state, producing useful metrics, and explaining engineering decisions clearly.

Please submit a solution that you can confidently explain, debug, and change. A focused, reliable solution is more valuable than a large or overly complex one.

## At a glance

- Submission window: one week elapsed time, not a week of continuous work
- Suggested effort: 6–8 focused hours
- Language: Python 3.12
- Data store: DuckDB
- Tests: pytest
- Required supported runtime and review path: Docker Compose
- Optional convenience: local Python setup
- Cloud account: not required

Read the [acceptance criteria](docs/acceptance-criteria.md) for exact required
schemas, audit definitions, failure behavior, and fixture examples. They form the
public contract together with this README.

## Time and accommodations

Use a suggested timebox of 6–8 focused hours within the one-week submission window.
Document unfinished work, priorities, and next steps rather than exceeding that
timebox. Extra time and optional bonuses are not a substitute for core correctness;
incomplete work with clear prioritization can still provide useful evidence.

Contact the hiring team if you need deadline or accessibility accommodations.
Requesting or using accommodations carries no penalty.

## The scenario

The Booking Operations team receives incremental CSV exports from a booking system. Operations needs two reliable outputs:

1. The latest valid state of every booking.
2. Daily pickup and drop-off counts by depot.

The source occasionally sends duplicate or out-of-order records, and some records are invalid. Your pipeline must handle these conditions without corrupting current state or inflating the report.

## Input data

The deterministic sample files are in [`data/`](data/):

- `depots.csv`
- `bookings_01.csv`
- `bookings_02.csv`
- `bookings_03.csv`

`depots.csv` and the supplied `bookings_01.csv`, `bookings_02.csv`, and
`bookings_03.csv` are immutable and must remain in the input directory.
Do not move, rename, edit, or delete them.

Process every file matching `bookings_*.csv` in ascending filename order. Do not rely on filesystem iteration order.
Every run may reread all source files. File checkpointing or stateful tracking of
consumed files is not required. All four supplied files remain required on every run.

### Depot columns

```text
depot_id,depot_name,country
```

### Booking columns

```text
booking_id,pickup_depot_id,dropoff_depot_id,pickup_at,dropoff_at,status,updated_at
```

## Required behavior

Treat the following rules as the source contract:

- Each booking CSV is an incremental export containing new or changed bookings. It is not a full snapshot.
- `booking_id` identifies a booking.
- `updated_at` determines the source version of a booking.
- A newer valid version replaces an older valid version.
- An older valid version that arrives later must not overwrite a newer version.
- Exact duplicate rows may occur. They must not create duplicate current records or inflate metrics.
- The supplied core data does not contain differing records with the same `booking_id` and `updated_at`. Document a deterministic policy for that conflict.
- Cancellations arrive as records whose `status` is `cancelled`.
- A booking missing from a later file has not been deleted.
- An invalid update must be rejected and must not replace an existing valid booking, even if its `updated_at` appears newer.
- Input timestamps are ISO 8601 UTC timestamps. Accept UTC expressed as `Z` or an explicit `+00:00` offset and normalize consistently. Reject timestamps with any other offset (for example `+02:00`) or with no offset.
- The only accepted statuses are `confirmed`, `completed`, and `cancelled`.

Validate each booking row before applying source-version rules. At minimum, reject a row when:

- `booking_id` is missing.
- Either depot ID is not present in `depots.csv`.
- `pickup_at`, `dropoff_at`, or `updated_at` is not a valid ISO 8601 UTC timestamp.
- `pickup_at` is later than or equal to `dropoff_at`.
- `status` is not supported.

You may add sensible validation rules or choose how to report multiple failures on one row. Document the policy so that a reviewer can understand and reproduce it.

### Business rejection versus fatal source errors

Booking business-rule failures belong in `rejected_records` and do not fail the run.
Missing any supplied required file is a fatal source-structure error.
Malformed CSV structure, row width, or quoting, invalid headers, duplicate
depot IDs, and blank depot IDs are also fatal.

Check the complete input for fatal source-structure errors **before any output
mutation**. On such an error, exit non-zero, add no successful-run audit rows,
and preserve all previously committed required models. Documenting this behavior
without implementing it is not sufficient.

Full disaster recovery for unexpected infrastructure failure or process termination
is not required. Document any remaining partial-write risk. A transaction is strong
protection, but the required behavior is fatal-source preflight before output mutation.

## Operational report

Build the report from current-state bookings whose status is `confirmed` or `completed`. Exclude `cancelled` bookings.

- Count a pickup on the UTC calendar date of `pickup_at`, at `pickup_depot_id`.
- Count a drop-off independently on the UTC calendar date of `dropoff_at`, at `dropoff_depot_id`.
- Include only date and depot combinations that have at least one pickup or one drop-off.
- Include both `pickup_count` and `dropoff_count` and use zero when the other event has no count for that date and depot.

For example, consider a hypothetical booking whose confirmed version has `updated_at=2026-01-01T10:00:00Z`. A cancellation with `updated_at=2026-01-03T10:00:00Z` replaces it. If a confirmed version with `updated_at=2026-01-02T10:00:00Z` then arrives in a later file, it is stale: the cancellation remains current and that booking contributes no report events.

## Required DuckDB models

Create the required models in the default `main` schema of the configured DuckDB
database. They must remain queryable after the process exits.

- `depots`: persisted table containing the loaded depot reference data.
- `current_bookings`: persisted table or view with one latest valid row per `booking_id`, including cancelled bookings.
- `rejected_records`: persisted table with stable source-row identity, lossless raw strings, and understandable reasons.
- `processing_audit`: persisted table with one row per successful execution and booking file.
- `daily_operations_report`: persisted table or view following the report rules above.

The [model contract](docs/acceptance-criteria.md#required-model-contract) specifies
every required column, type, and key. All listed required columns are NOT NULL.
Extra useful columns are allowed. Logical keys must be unique. Enforce physical
primary keys when a model is a table. Foreign-key constraints are not required.

Required views must query persisted DuckDB data only. After execution, they must
not read source CSV files directly or through another view. No live `read_csv*`
or other external-file dependency is allowed in a required view.

Repeated successful runs must leave logical current bookings, logical rejected
records, and the daily report identical. Successful audit rows may append per execution.
Rejection identity is `<source_filename>:<1-based data-row number>`, excluding the header.

### Audit contract

`processing_audit` has an exact field-by-field contract: fixed count meanings, a
required classification order (validate, then duplicate, then stale, then applied),
and two per-file accounting identities that must hold on every execution. The
precise definitions, the expected first-run totals, and the repeat-run rules are
specified once in the
[acceptance criteria](docs/acceptance-criteria.md#processing_audit). Read them
before implementing; they are graded as written.

## What to build

Complete the starter code so that it:

1. Provides a command-line entry point with configurable `--input-dir` and `--database` paths.
2. Loads the depot reference and all booking batches.
3. Applies validation before current-state version selection.
4. Produces the required DuckDB models.
5. Processes files repeatably in the required order and is idempotent when rerun against the same database.
6. Includes meaningful pytest tests written by you. The supplied smoke test only verifies the environment and does not count as a business-behavior test.
7. Works with the documented Docker Compose build, CLI, test, and pipeline commands. Docker Compose is the required supported review path; a local Python workflow is optional, not a submission requirement.
8. Implements at least one meaningful core transformation in SQL: current-state selection or daily report aggregation.

The SQL requirement demonstrates SQL proficiency; it does not require a particular
reference architecture. You choose which of those core transformations uses SQL
and may organize the remaining internals as you see fit. You are not required to
preserve the starter function boundaries. Table creation or simple output queries
alone do not satisfy the core SQL requirement.

## Your README

Add a solution section to this README, or add a clearly linked solution document. Include:

- Setup and exact run commands.
- Design decisions and assumptions.
- How you implement the required audit definitions and accounting identities.
- How you handle idempotency, duplicates, stale records, invalid updates, and same-version conflicts.
- Test strategy and commands.
- Known limitations, unfinished work, and priorities for next steps.
- Fatal-source preflight and preservation behavior, plus any remaining partial-write risk from unexpected failures.
- Improvements you would make for a production workload.

Environment commands are provided in [`starter-kit/README.md`](starter-kit/README.md).

## Use of AI

You are welcome to use AI tools. AI use is not required. The same suggested 6–8
focused-hour timebox applies with or without AI; the one-week window is elapsed
submission time, not expected continuous labor.

You remain responsible for everything you submit. You must be able to understand, validate, explain, debug, and modify all code, SQL, tests, and documentation in your repository. During the presentation, you may be asked to diagnose behavior or make a small change.

## What we evaluate

We value:

- Correctness of validation, versioning, and metrics.
- Idempotent, repeatable behavior.
- Clear Python and SQL, including at least one core SQL transformation as described above.
- Meaningful tests around important edge cases.
- Explicit assumptions and sensible trade-offs.
- Ownership and understanding of the submitted work.

We do not reward unnecessary architecture, and we do not require 100% test coverage.

## Bonus ideas

Full core score does **not** require cloud deployment, workflow orchestration,
file checkpointing, CI, production observability, historical/SCD models, or
implemented disaster recovery. You may describe production improvements instead.
The implemented fatal-source preflight rule remains core behavior.

The following are optional bonuses only. Complete the core behavior before considering them, and stay within the suggested timebox. Bonuses do not compensate for incorrect core behavior:

- Booking history or SCD modeling.
- Workflow orchestration.
- Continuous integration.
- Data observability or richer data-quality reporting.
- A cloud or AWS production design.

Cloud services are not required. Keep the submitted solution runnable and
reviewable with Docker Compose without a cloud account.

## Submission

- Create a private GitHub repository containing your solution.
- Invite the reviewer specified in your interview instructions.
- Submit within the agreed one-week window.
- Do not include credentials, secrets, or personal customer data.

## Presentation

Plan for approximately 30 minutes:

- 10 minutes: architecture and code walkthrough.
- 10 minutes: correctness and idempotency demonstration.
- 10 minutes: a small change or debugging discussion.

Good luck, and have fun building it.
