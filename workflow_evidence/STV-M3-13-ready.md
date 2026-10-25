# STV-M3-13 next implementation slice

**Status:** Readiness brief for #103. This is not implementation evidence or operator acceptance.

**Issue:** #631 prepares the next bounded slice. Parent #103 stays open and BLOCKED.

**Repository:** `djh00t/steveproxy`. The `djh00t/steve` URL redirects to this repository.

**Inspected base:** `5dbc4b88960a88979456e12992bab8af5968fea2` on `main`, 2026-10-25.

## Accepted inputs

- #102 accepted `docs/contracts/stv-m3-40.md` at `a8c9b729f7067272a1622bb4f4a9cf637efa13b0`; merge `4472ca59896465fcf27b0d1df1d5218552d80efd`.
- #72 is closed. Its real-process harness is present in `tests/e2e.rs`.
- #597 added ledger-prefix rejection.
- #599 added ledger-column validation.
- #611 added foundation-column validation and preserves an empty ledger when table shape is invalid.
- PR #624 merged the repository dispatch inventory. The live inventory gives #103 the owned paths `src/storage/db.rs::migrate_sqlite` and `tests/e2e.rs`, requires one writer, and classifies the implementation command as producer-introduced.

## Current implementation

`src/storage/db.rs:544-624` already:

- creates and validates the v1 migration ledger;
- rejects a malformed ledger prefix;
- creates v1 in one SQLite transaction;
- validates `steve_background_events` columns before committing v1.

The remaining defect is in `src/storage/db.rs:585-612`. When the ledger is empty and a correctly shaped `steve_background_events` table already exists, `CREATE TABLE IF NOT EXISTS` leaves the table in place. The code then inserts the v1 ledger row and treats the database as fresh. The accepted contract says a ledgerless database is fresh only when no Steve application table exists. A foundation table without its v1 row must fail visibly and remain unchanged.

The existing `sqlite_rejects_foundation_column_drift` scenario covers invalid foundation columns with an empty ledger. It does not cover a valid foundation table with data and an empty ledger.

## Required child issue

**Issue-tracker prerequisite (orchestrator-owned):** Before dispatch, the orchestrator creates and accepts child issue `[STV-M3-13D] Reject SQLite foundation tables without a migration ledger` under #103, using this brief's behavior and acceptance as its scope. Issue creation is not part of this implementation package or its owned paths. Do not create a branch or begin implementation until the child issue exists and the orchestrator has assigned this brief. If the child issue is not created, report BLOCKED; do not publish an issue or broaden scope yourself.

**Kind:** one runtime guard.

**Active effort target:** 5-10 minutes, excluding build and review waits.

**Owned paths:** `src/storage/db.rs::migrate_sqlite` and `tests/e2e.rs`.

**Single writer:** Yes. Serialize both files with every other migration change.

**Dependencies:** Accepted #102 contract and accepted or merged #597, #599, and #611 slices, all present on the inspected base.

## Required behaviour

In the `current < 1` path, check `sqlite_master` for an existing `steve_background_events` table before `CREATE TABLE IF NOT EXISTS`. If the table exists, fail with `SQLite foundation table exists without a migration ledger`.

A fresh database still creates both v1 tables and records `(1, "m0_foundation")`.

An existing ledgerless foundation table still causes startup to fail before readiness. The failed startup must leave the ledger empty and preserve every application row.

Use one `EXISTS` query and `anyhow::bail!`. Do not add a migration framework, configuration, dependency, repair command, or automatic adoption.

## Acceptance scenario

The named scenario's primary acceptance is the existing ledgerless-table rejection and preservation assertions below. Do not treat this child as qualification of the full `sqlite_migration_runner` command or as completion of #103. The fresh v0 -> v1 positive case remains part of the composed `sqlite_migration_runner` acceptance on #103 and is omitted from this child.

Add one real-process test named `sqlite_rejects_ledgerless_foundation`.

Create the exact shipped table shapes from the SQLite DDL at `src/storage/db.rs:546-553` and `588-593`, insert one sentinel row, and confirm the ledger has zero rows before starting Steve.

Given that temporary SQLite database, when Steve starts, then it exits before `listeners_ready` with an error that contains exactly `SQLite foundation table exists without a migration ledger`, plus the harness's `Steve exited` wrapper.

After the failed start, query SQLite directly and assert that:

- `steve_schema_migrations` still has zero rows;
- `steve_background_events` still has the sentinel row with all four values unchanged.

Run:

```text
cargo test --test e2e --all-features sqlite_rejects_ledgerless_foundation -- --exact --nocapture
```

The command must select exactly one test. Before the implementation it must fail at the ledgerless readiness assertion. After the implementation it must pass. A zero-test selection is failure.

## Mutation evidence

Temporarily bypass the new existing-foundation-table rejection (for example, force its `EXISTS` result to false). The `sqlite_rejects_ledgerless_foundation` test must then fail at its expected startup/readiness-rejection assertion. Restore the guard and rerun the exact command.

Record the mutation outcome as caught, survived, equivalent, unviable, or timeout. Do not invent a broader mutation score.

## Checks and delivery

- Run the named test and confirm one test ran.
- Run `make check`.
- Run `git diff --check`.
- Commit with `fix(storage): reject SQLite tables without a ledger`.
- Open a ready-for-review PR that uses `Fixes #<child issue>` and links #631 and #103.
- Report the exact head SHA, local results, and hosted check state.
- Do not merge, approve, deploy, or close #103.

The child command is `sqlite_rejects_ledgerless_foundation`; it is not the inventory's `sqlite_migration_runner` acceptance command and does not mark #103 complete. Before #103 is ready for acceptance, update/compose the producer's `sqlite_migration_runner` test so it includes the relevant ledgerless rejection (and fresh v0 -> v1 positive case) and run the exact inventory command selecting exactly one test. Keep the inventory command producer-introduced until verified on its candidate branch; do not mark it verified from this child alone.

## Boundaries

This slice owns only the ledgerless-foundation guard.

The following accepted-contract work remains in #103 or later child issues:

- ledger constraints and triggers;
- one atomic invocation covering lock, validation, migration, and ledger write;
- bounded SQLite contention and deadline;
- ambiguous-commit verification;
- full `sqlite_migration_runner` qualification;
- PostgreSQL parity in #104.

Do not infer full migration-runner completion from this slice.
