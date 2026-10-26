# Changelog

All notable changes to **Feather** will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.2.3] - 2026-10-27

### Added
- **String and UUID Grouped Relation Loading (`relations.nox`):**
  - Added `RelationLoader.load_has_many_grouped_str[C]` and `load_relation_has_many_grouped_str[C]` returning `dict[str, list[C]]` mapping string and UUID primary keys to child record lists.
- **Configurable Connection Liveness Ping (`pool.nox`):**
  - Added `ping_on_acquire: bool = True` to `PoolConfig`, `create_pool`, and `create_sqlite_pool` enabling optional fast-path acquire without round-trip health checks in latency-critical environments while retaining full capability to isolate poisoned connections.
- **Pre-Rollback Checksum Verification in `migrate_down()` (`migrations.nox`):**
  - `migrate_down()` now executes `verify_checksums()` prior to executing any rollback SQL, ensuring tampered migration files cannot execute arbitrary unverified down migrations.
- **Dedicated Transaction Lifecycle and Atomic Migration Verification Test Suites:**
  - `tests/integration/transaction_lifecycle_test.nox` verifying clean connection releases, uncommitted transaction auto-rollbacks, rollback failure isolation, and configurable health checks.
  - `tests/integration/migration_rollback_verify_test.nox` verifying atomic multi-migration legacy upgrades and rollback aborts upon checksum divergence.

### Changed & Hardened
- **Strict Conditional Rollback on Connection Release (`pool.nox`, `session.nox`):**
  - `release_connection()` only triggers `ROLLBACK` when `s.in_transaction == True`. Clean connections (committed or idle) are immediately returned to the idle pool without executing redundant rollbacks or generating benign driver errors.
  - Raw SQL inspection in `Session.execute_raw` now tracks `in_transaction = True` on `BEGIN`/`START TRANSACTION` and `False` on `COMMIT`/`ROLLBACK`.
- **Transaction Abort State Preservation on Failed Operations (`transaction.nox`):**
  - `Transaction.commit()` and `rollback()` only clear `s.in_transaction` upon successful completion. Failed commits or rollbacks preserve `in_transaction = True` so connection pool release reliably cleans up or discards the compromised connection.
- **Atomic Legacy Migration Checksum Upgrades (`migrations.nox`):**
  - `MigrationManager.verify_checksums()` now executes in two distinct phases: in-memory verification across all migrations first, followed by a single transactional batch update for all legacy checksums (`UPDATE _feather_migrations SET checksum = ? WHERE version = ?`). If any migration in the suite fails verification, the transaction aborts and no checksums are partially upgraded.

## [0.2.2] - 2026-10-27

### Added
- **Poisoned / Severed Connection Isolation (`pool.nox`):**
  - Detects broken, poisoned, or severed database sockets during uncommitted transaction rollback on connection release.
  - Poisoned connections are permanently closed (`fc.close()`) and discarded, preventing recycled bad connections from corrupting subsequent callers.
- **Connection Health Checks on Acquire (`pool.nox`):**
  - Executes a lightweight liveness query (`SELECT 1`) on idle connections prior to lease issuance; dead connections are cleanly disposed of and replaced with fresh connections from the factory.
- **Graceful vs Force Pool Shutdown (`pool.nox`):**
  - `close_all(force: bool = False)`, `close_graceful()`, and `close_force()`.
  - Default graceful shutdown marks the pool as closed to new requests while allowing in-flight leased sessions to complete their queries before closing upon release.
- **Migration Checksum Backward Compatibility (`migrations.nox`):**
  - Seamless automatic upgrade path from v0.2.0 (8-character DJB2) to modern SHA-256 (64 characters).
  - Automatically validates legacy checksums with `compute_legacy_migration_checksum()` and upgrades the database tracking record in `_feather_migrations` to SHA-256 on the fly without breaking existing databases.
- **Grouped Batched Relation Loading (`relations.nox`):**
  - `RelationLoader.load_has_many_grouped[C]` and `load_relation_has_many_grouped[C]` returning `dict[int, list[C]]` mapping parent primary keys directly to their child records.
- **Parameter Chunking for Large IN Clauses (`relations.nox`):**
  - Added `BATCH_CHUNK_SIZE = 500` chunking in `load_has_many_batch` and `load_belongs_to_batch` to avoid database engine parameter limit violations.

### Changed & Hardened
- **Transaction State Tracking on Session (`session.nox`, `transaction.nox`):**
  - Added `Session.in_transaction: bool` managed directly by `Transaction.begin()`, `commit()`, and `rollback()`.
  - Avoids redundant rollback executions on clean connections during pool release.

## [0.2.1] - 2026-10-27

### Added
- **Batched Relation Eager Loading (`relations.nox`):**
  - `RelationLoader.load_has_many_batch[C](child_model, fk_col, parent_pks)` and `load_belongs_to_batch[P](parent_model, pk_col, fks)` querying multiple parent/foreign records in a single `IN (?, ?, ...)` query, completely eliminating N+1 query patterns.
  - Added relation helper wrappers: `load_relation_has_many_batch[C]` and `load_relation_belongs_to_batch[P]`.
  - Immediate empty-list return safeguard avoiding redundant database roundtrips.
- **Parenthesized Condition Grouping & Expression AST (`query.nox`):**
  - `SelectQuery.where_group(grp: WhereClause, logic: int = 1)` and `or_where_group(grp: WhereClause)` supporting arbitrary parentheses grouping such as `(role = 'admin' OR role = 'moderator') AND is_active = true`.
  - `WhereClause.add_group(grp: WhereClause, logic: int = 1)` enabling nested clause structures.
  - `ParamCollector` abstraction for safely collecting parameters across recursive where clause compilation while respecting compiler limitations.
- **Dialect-Aware Migration Table DDL (`dialects.nox`, `migrations.nox`):**
  - Added `Dialect.migration_table_sql()` ensuring valid DDL across all engines: `INT AUTO_INCREMENT PRIMARY KEY` on MySQL, `SERIAL PRIMARY KEY` on PostgreSQL, and `INTEGER PRIMARY KEY AUTOINCREMENT` on SQLite.

### Changed & Hardened
- **Connection Pool Lease Security & Invariant Maintenance (`pool.nox`, `session.nox`):**
  - Unique `pool_id` identification and monotonic per-acquisition `lease_id` tracking.
  - Double-release prevention: throws `PoolError` if a session is released more than once.
  - Foreign-pool release prevention: throws `PoolError` if attempting to release a session into a different pool.
  - Session invalidation on release: sets `is_released = True`, causing subsequent database operations on the released session to raise `ConnectionError`.
  - Automatic rollback of uncommitted transactions upon release to prevent transaction leakage.
  - Preserved pool capacity invariant: `total = len(idle_conns) + len(active_leases) <= max_connections`.
- **Cryptographic SHA-256 Migration Checksums (`migrations.nox`):**
  - Replaced non-cryptographic hash with standard `nox.crypto.sha256` yielding canonical 64-character hex digests.
  - Checksum computation now encompasses migration version, name, all `up_sqls`, and all `down_sqls` for tamper protection.
- **Strict `insert_many` Column Uniformity Validation (`session.nox`):**
  - Validates that every row in batch insertion possesses the exact column set derived from the initial row; raises `QueryBuildError` on missing or extraneous keys, preventing silent SQL NULL insertion.
- **Aggregate Column Identifier Validation (`query.nox`):**
  - Validates inner column identifiers inside `COUNT(...)`, `SUM(...)`, `AVG(...)`, `MIN(...)`, and `MAX(...)` in `SelectQuery.__init__` and `compile`, rejecting SQL injection fragments with `IdentifierError`.
- **Composite Primary Key Validation (`metadata.nox`):**
  - Rejects `is_auto_increment=True` on tables with multi-column composite primary keys at definition time with `QueryBuildError`.

## [0.2.0] - 2026-10-27

### Added
- **Lossless Type Codecs (`types.nox`, `value.nox`, `mapper.nox`):**
  - `Decimal` / `Money`: Fixed-point and arbitrary precision monetary value codec backed by string representation to prevent float rounding errors.
  - `DateTime`: ISO-8601 UTC parser and formatter (`parse_iso_datetime`) with year, month, day, hour, minute, second components.
  - `UUID`: RFC-4122 compliant 36-character canonical hex validator and formatter (`uuid_parse`, `is_valid_uuid`).
  - Added value constructors: `val_decimal`, `val_datetime`, `val_uuid`.
  - Added row mapper decoders: `row_get_decimal`, `row_get_opt_decimal`, `row_get_datetime`, `row_get_opt_datetime`, `row_get_uuid`, `row_get_opt_uuid`.
- **Batch Insertion (`session.nox`, `dialects.nox`):**
  - `session.insert_many(tbl: Table, rows: list[dict[str, Value]]) -> int`: Single-query multi-row `VALUES (?, ?), (?, ?)...` generator.
  - Dialect-aware parameter index compilation (`?` for SQLite/MySQL, dynamic `$1..$N` for PostgreSQL).
  - Empty-list safeguard returning 0 immediately without executing database queries.
- **SQL Aggregates, GROUP BY & HAVING AST (`query.nox`):**
  - Aggregate functions: `count_all()`, `count_col(col)`, `sum_col(col)`, `avg_col(col)`, `min_col(col)`, `max_col(col)`.
  - `SelectQuery.group_by(cols: list[str])` for grouping results with identifier validation.
  - `SelectQuery.having(pred: Predicate)` with parameterized predicate conditions (`HAVING count > ?`).
- **Composite Primary Key Support (`metadata.nox`, `session.nox`, `dialects.nox`):**
  - Multi-column primary key definitions in `Table` via `primary_keys: list[str]`.
  - Dialect DDL generation with table-level `PRIMARY KEY ("col1", "col2")` constraint and `NOT NULL` column definitions.
  - Composite CRUD helpers: `update_by_pk(tbl, pk_values, values)`, `delete_by_pk(tbl, pk_values)`, and `find_by_pk(model, pk_values)`.
- **Migration Checksum & Tamper Protection (`migrations.nox`):**
  - Deterministic 8-hex-digit checksum computation for all migration SQL statements (`compute_migration_checksum`).
  - Tracking table `_feather_migrations` stores recorded `checksum TEXT NOT NULL`.
  - Verification on `migrate_up` aborts with `MigrationError` if an already applied migration was altered in code.
- **Many-to-Many Relationships (`relations.nox`):**
  - `many_to_many(name, source_table, target_table, through_table, source_fk, target_fk, target_pk)` relation definition.
  - `RelationLoader.load_many_to_many` and `RelationLoader.load_relation_many_to_many` eager loading via inner join query through pivot table.
- **Upsert Operations (`session.nox`, `dialects.nox`):**
  - `session.upsert(tbl, values, conflict_target, update_values) -> int`.
  - Multi-dialect syntax: `ON CONFLICT (...) DO UPDATE SET ...` or `DO NOTHING` on SQLite and PostgreSQL, and `ON DUPLICATE KEY UPDATE ...` on MySQL.
- **CLI Migration & Verification Tool (`main.nox`):**
  - Subcommands: `feather demo`, `feather migrate status [db]`, `feather migrate up [db]`, `feather migrate down [ver] [db]`.

## [0.1.0] - 2026-10-27

### Added
- **Core Model & Schema System:**
  - Declarative table schema metadata (`Table`, `Column`) with identifier syntax validation (`validate_identifier`).
  - Column builders: `col_int`, `col_str`, `col_float`, `col_bool`.
  - Generic `Model[T]` pairing `Table` with typed `decoder: (Row) -> T` and `encoder: (T) -> dict[str, Value]`.
  - Type-safe row getters with SQL `NULL` optionals (`row_get_int`, `row_get_str`, `row_get_float`, `row_get_bool`, `row_get_opt_int`, `row_get_opt_str`, etc.).
- **Type-Safe Value Representation:**
  - `Value` union (`val_int`, `val_str`, `val_float`, `val_bool`, `val_null`).
  - Zero-based parameter binder for single-use Nox `Statement` handles.
  - Formatted debug value logging (`format_bound_values`).
- **Multi-Dialect Query Engine:**
  - `Dialect` abstraction supporting SQLite, PostgreSQL, and MySQL.
  - Dialect-aware identifier quotation (`"` for SQLite/Postgres, `` ` `` for MySQL).
  - Dialect-aware parameter placeholders (`?` for SQLite/MySQL, `$1..$N` for PostgreSQL).
  - Dialect DDL generation with auto-increment mapping (`INTEGER PRIMARY KEY AUTOINCREMENT`, `SERIAL PRIMARY KEY`, `INT AUTO_INCREMENT PRIMARY KEY`).
- **AST Query Builder:**
  - Fluent `SelectQuery` AST builder supporting `SELECT`, `WHERE`, `ORDER BY`, `LIMIT`, `OFFSET`, `INNER JOIN`, and `LEFT JOIN`.
  - Predicates: `eq`, `neq`, `gt`, `gte`, `lt`, `lte`, `like`, `is_null`, `is_not_null`, `is_in`.
  - Invariant security: Empty `is_in([], ...)` lists safely compile to `1 = 0`.
- **Session & Unit of Work:**
  - `Session` coordinating connection execution, DDL, parameterized queries, and typed CRUD (`insert`, `find_all`, `find_by_id`, `find_where`, `update_by_id`, `delete_by_id`, `count`).
  - Protection against accidental table wipes on unfiltered `update` and `delete`.
  - Parameterized raw SQL execution escape hatch (`execute_raw`, `query_raw`).
- **Explicit Transactions:**
  - `Transaction` lifecycle tracking states: `IDLE`, `ACTIVE`, `COMMITTED`, `ROLLED_BACK`, `FAILED`.
  - Automatic rollback on exception in transaction block scopes.
- **Connection Pooling:**
  - `ConnectionPool` with bounded capacity, `acquire()`, `release_connection()`, and `close_all()`.
- **Schema Migrations:**
  - `MigrationManager` with transactional `migrate_up()` and `migrate_down()` execution.
  - `_feather_migrations` version tracking table.
  - Migration status reporter (`status()`).
- **Relationships:**
  - Declarative foreign-key relationship metadata (`has_one`, `has_many`, `belongs_to`).
  - `RelationLoader` for explicit, transparent eager loading without N+1 queries.
- **Testing & Documentation:**
  - 12 comprehensive unit and SQLite in-memory integration test suites passing 100%.
  - 8 executable examples in `examples/`.
  - Full documentation suite in `docs/`.
