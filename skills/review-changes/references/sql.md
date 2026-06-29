# SQL / Database Script Review Criteria

Applies to `.sql` files and Liquibase changelogs (`.xml`/`.yaml`/`.json` containing `databaseChangeLog`, or under a `changelogs/`/`liquibase` path). Covers DDL (new tables/columns/indexes), DML migrations/backfills, and ad-hoc ops scripts.

**Context first:** identify the engine and dialect (LegalMatch = **MySQL / InnoDB**), whether the script is **schema (DDL)**, **data migration / backfill (DML)**, or **ops/one-off**, and whether it ships via **Liquibase** (auto-run at deploy) or is **hand-run by ops per environment**. Read the surrounding schema — the *existing* table definitions, indexes, charset, and naming conventions are the baseline a change must fit. When the script touches a real table, verify your assumptions against the live schema (`information_schema.COLUMNS` / `.STATISTICS` / `.TABLES`) rather than trusting the script's comments.

Evaluate against the categories below. Report only actionable findings — skip categories where the change is clean.

### Indexing — new tables (REQUIRED CHECK)

For every new table or new column, decide what should be indexed and confirm the script creates it:

- **Primary key** present and sensible (surrogate `BIGINT AUTO_INCREMENT` vs natural key). InnoDB clusters on the PK — a poor PK choice penalizes every secondary index.
- **Foreign-key columns** indexed. InnoDB requires an index on the referencing column; confirm one exists (the FK constraint creates it only if absent).
- **Columns the application will filter, join, sort, or group by** are indexed. Walk the access patterns the feature introduces (the new repository methods / queries) and confirm each `WHERE` / `JOIN` / `ORDER BY` / `GROUP BY` column is covered.
- **Composite index column order**: leading column must be the one used alone or with equality; order by selectivity and by the query's predicate shape. A composite `(a, b)` already serves `a`-only lookups — a separate single-column index on `a` is redundant.
- **Don't over-index**: every index is write amplification (insert/update/delete + redo/undo + replication). Flag indexes on low-cardinality columns (booleans, status flags with 2–3 values) that the optimizer will ignore, and redundant/duplicate indexes (prefix already covered by a composite).
- **Uniqueness**: should a `UNIQUE` constraint enforce a business key? Note MySQL allows multiple NULLs in a unique index (NULL-tolerant) — useful for add-column-then-backfill, but it means uniqueness isn't enforced for NULL rows.

### Indexing — backfill / migration scripts (REQUIRED CHECK)

The migration query has its *own* performance profile, separate from the app's:

- **Every column in the backfill's `JOIN`/`WHERE`/filter must be indexed on the tables being scanned — including source and staging tables.** If a filter or join column is **not** indexed on the source/staging table, the migration degrades to a nested-loop near-cross-product (`O(N×M)`): instant on a staging dataset, 30+ minutes at prod scale. Either add the index (temporary, dropped after — or permanent if the app needs it too) or explicitly account for the full-scan cost and call it out.
- **Verify the join is actually index-served**, not just that an index exists. Things that silently disable an index on the join/filter column:
  - **Charset/collation mismatch** between joined columns (e.g. a `latin1` staging table joined to a `utf8mb4` target). MySQL converts the indexed column to compare → index unusable → full scan. Match the staging table's charset/collation to the target explicitly; never rely on the session/DB default for a hand-created staging table.
  - **Implicit type conversion** (string column vs numeric literal, `DATETIME` vs `VARCHAR`).
  - **A function or expression wrapping the indexed column** (`DATE(col) = ...`, `LOWER(col) = ...`, leading-wildcard `LIKE '%x'`).
  - **`<=>` (NULL-safe equal) and `OR`** — often not index-served; prefer plain `=` where NULLs aren't possible.
- **Confirm join order / drive direction**: the table whose index serves the join must be the *probed* (inner) side. When in doubt, `EXPLAIN` the exact statement (or `STRAIGHT_JOIN` to pin order). Freshly loaded staging tables have no statistics — recommend `ANALYZE TABLE` so the optimizer doesn't pick a cross-product plan.
- **Drop redundant predicates**: if a `(colA, colB)` pair is already unique, an extra `AND expensive_text_col <=> ...` adds cost (often on an unindexable `TEXT`/large `VARCHAR`) for no disambiguation.

### Batching, Locking & Transaction Scope

- **Large `UPDATE`/`DELETE`/`INSERT...SELECT` must be chunked** (keyset/`LIMIT` loop), not one monster statement. A single multi-million-row DML holds locks for its whole duration, bloats undo/redo and the binlog, and spikes replica lag.
- **Lock scope**: what rows/gaps does the statement lock, and for how long? Under InnoDB REPEATABLE READ, range scans take gap locks that block concurrent inserts. Prefer narrow, indexed predicates; run heavy migrations in low-traffic windows.
- **DDL is (mostly) non-transactional in MySQL** — each DDL statement implicitly commits and can't be rolled back inside a transaction. Don't assume a `BEGIN ... ROLLBACK` will undo an `ALTER`.
- **Online DDL**: for `ALTER TABLE` on a large table, specify/verify `ALGORITHM=INPLACE, LOCK=NONE` is supported for that change; if it forces a `COPY` (table rebuild, full lock), call it out and consider `pt-online-schema-change` / `gh-ost`. On RDS, factor in the rebuild's storage/IOPS impact.

### Idempotency & Re-runnability

- **Can the script be safely re-run** after a partial failure? Backfills should have a resume-friendly predicate (`WHERE target_col IS NULL`) so a re-run only touches unfinished rows.
- **Liquibase preconditions** make changesets idempotent: `tableExists` / `columnExists` / `indexExists` with `onFail="MARK_RAN"` so a redeploy or a manually-applied change doesn't fail or double-apply.
- **Guard destructive steps**: `DROP`/`TRUNCATE` of a staging/temp table should run only after the dependent step succeeded, and ideally `DROP TABLE IF EXISTS`.

### Liquibase Conventions

- **One logical change per `changeSet`**; never edit a changeSet that has already been applied anywhere — Liquibase validates by checksum and a changed checksum errors on the next run. Add a new changeSet instead.
- **`rollback`** defined where feasible (or an explicit empty rollback with rationale) so a release can be reverted.
- **Separate big data backfills from the auto-run changelog.** Schema (add column + index) belongs in Liquibase (runs at deploy); a heavy multi-environment data reconciliation is better as an ops-run script executed per env after deploy — keep the deploy fast and let the DBA control the backfill's timing/locking. (This is the established LegalMatch pattern.)
- **Stable `id`/`author`/`logicalFilePath`** and correct file ordering (numeric prefix convention). `contexts`/`labels` for environment-specific changes.

### Correctness & SQL Semantics

- **`UPDATE`/`DELETE` always scoped** by a `WHERE` (or deliberately, with a comment) — a missing/over-broad `WHERE` is catastrophic and irreversible.
- **NULL handling**: three-valued logic — `= NULL` never matches (use `IS NULL`); `NOT IN (subquery with NULLs)` returns no rows; use `<=>` only intentionally; `COALESCE`/`IFNULL` at boundaries.
- **JOIN fan-out**: a one-to-many join multiplies rows and silently inflates `SUM`/`COUNT`/`AVG`. Confirm cardinality; use `EXISTS`/`IN` or pre-aggregation where you don't want the multiplication.
- **LEFT JOIN turned INNER**: a `WHERE` predicate on the right-hand table of a `LEFT JOIN` (other than `IS NULL`) silently drops the unmatched rows — move it to the `ON` clause if that wasn't intended.
- **Determinism for replication**: row-based binlog is safer, but non-deterministic functions (`NOW()`, `RAND()`, `UUID()`) and `INSERT ... ON DUPLICATE` with auto-increment can diverge under statement-based replication — be explicit.
- **`SELECT *`** in views/migrations is brittle — name columns.

### Data Types, Charset & Schema Hygiene

- **Right-sized, correct types**: `BIGINT` for ids that join to `BIGINT` PKs (type mismatch on a join key disables the index); `DECIMAL` for money (never `FLOAT`/`DOUBLE`); `DATETIME` vs `TIMESTAMP` (range + timezone behavior — LegalMatch DBs run `America/Los_Angeles`); appropriate `VARCHAR` lengths; `TINYINT(1)`/boolean conventions consistent with the schema.
- **Charset/collation explicit and consistent** with the surrounding schema (LegalMatch standard is `utf8mb4` / `utf8mb4_0900_ai_ci`). A new table created without an explicit charset inherits a default that may not match its neighbors — and a join-key collation mismatch is both a correctness risk ("illegal mix of collations") and a silent performance cliff (see backfill indexing above).
- **Nullability & defaults** deliberate: `NOT NULL` with a sane `DEFAULT` where the column is required; adding a `NOT NULL` column without a default to a populated table fails or forces an implicit default.
- **Naming conventions** match the existing schema (column prefixes, `idx_`/`fk_` naming, snake_case).

### Production Safety & Backward Compatibility (senior lens)

- **Additive-first / expand-contract**: during a rolling deploy, the schema must satisfy *both* the old and new app versions. Add nullable columns and new indexes first; backfill; switch reads/writes; only drop/rename old columns in a later release. A rename or type-narrowing in one shot breaks in-flight instances.
- **Blast radius & reversibility**: destructive or irreversible ops (`DROP`, `TRUNCATE`, type narrowing, `NOT NULL` tightening) need extra scrutiny, a backup/snapshot, and a rollback story.
- **Tested at prod scale, not just staging.** Row counts, data skew, and index/charset realities differ — the classic failure is a migration that's instant on `w4` and 30 minutes in prod. Estimate rows affected and expected lock/runtime, and hand the DBA an `EXPLAIN` + a runtime estimate.
- **Replica impact**: big transactions lag replicas (LegalMatch reads from `prod-slave`); chunk and pace accordingly.
- **Secrets / PII**: no credentials inline; mind PII when scripts dump to TSV/CSV or staging tables.
