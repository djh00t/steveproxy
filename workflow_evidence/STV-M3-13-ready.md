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

Create a child of #103 titled `[STV-M3-13D] Reject SQLite foundation tables without a migration ledger`. Do not create a work branch until that child issue exists.

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

Add one real-process test named `sqlite_rejects_ledgerless_foundation`.

Given a temporary SQLite database with the shipped foundation table shape, one sentinel row, and an empty migration ledger, when Steve starts, then it exits before readiness with the guard message.

After the failed start, query SQLite directly and assert that:

- `steve_schema_migrations` has zero rows;
- `steve_background_events` still has the sentinel row with all four values unchanged.

Also start Steve against a fresh temporary database and assert readiness plus ledger row `(1, "m0_foundation")`. This positive case proves the guard does not block first install.

Run:

```text
cargo test --test e2e --all-features sqlite_rejects_ledgerless_foundation -- --exact --nocapture
```

The command must select exactly one test. Before the implementation it must fail at the ledgerless readiness assertion. After the implementation it must pass. A zero-test selection is failure.

## Mutation evidence

Make the guard return false for an existing ledgerless foundation table. The named scenario must fail at the intended readiness assertion. Restore the guard and rerun the command.

Record the mutation outcome as caught, survived, equivalent, unviable, or timeout. Do not invent a broader mutation score.

## Checks and delivery

- Run the named test and confirm one test ran.
- Run `make check`.
- Run `git diff --check`.
- Commit with `fix(storage): reject SQLite tables without a ledger`.
- Open a ready-for-review PR that uses `Fixes #<child issue>` and links #631 and #103.
- Report the exact head SHA, local results, and hosted check state.
- Do not merge, approve, deploy, or close #103.

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
