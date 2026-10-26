# Feather API Reference (docs/api.md)

This document provides a comprehensive reference of the public classes, functions, and interfaces available in Feather.

## Core Session & Connection Management

### `connect_sqlite(path: str) -> Session`
Connects to an SQLite database file at `path` and returns a managed `Session`.

### `open_sqlite_memory() -> Session`
Convenience helper that opens an in-memory SQLite database (`:memory:`) and returns an active `Session`.

### `create_session(conn: DbConnection, dialect: Dialect, owns_conn: bool = True) -> Session`
Wraps any Nox `DbConnection` conforming connection with the given `Dialect`.

### `class Session`
Unit of work coordinating SQL execution, transactions, and CRUD.

- `create_table(tbl: Table) -> None`: Generates and executes DDL to create table.
- `drop_table(tbl: Table) -> None`: Drops table if it exists.
- `execute_raw(sql: str, params: list[Value]) -> int`: Executes parameterized DDL/DML, returning affected row count.
- `query_raw(sql: str, params: list[Value]) -> list[Row]`: Executes parameterized SQL query, returning rows.
- `insert(tbl: Table, values_dict: dict[str, Value]) -> int`: Inserts record, returning generated ID or affected count.
- `find_all[T](model: Model[T]) -> list[T]`: Queries all records and maps to typed model list.
- `find_by_id[T](model: Model[T], id_val: Value) -> T | None`: Finds single record by primary key.
- `find_where[T](model: Model[T], pred: Predicate) -> list[T]`: Finds records matching predicate.
- `execute_query[T](model: Model[T], q: SelectQuery) -> list[T]`: Compiles and executes query builder AST.
- `update(tbl: Table, values_dict: dict[str, Value], where_clause: WhereClause) -> int`: Updates records matching where clause.
- `update_by_id(tbl: Table, id_val: Value, values_dict: dict[str, Value]) -> int`: Updates single record by primary key.
- `delete(tbl: Table, where_clause: WhereClause) -> int`: Deletes records matching where clause.
- `delete_by_id(tbl: Table, id_val: Value) -> int`: Deletes single record by primary key.
- `count(tbl: Table) -> int`: Counts all rows in table.
- `count_where(tbl: Table, where_clause: WhereClause) -> int`: Counts rows matching condition.
- `close() -> None`: Closes session and underlying connection if owned.

---

## Model Metadata & Mapping

### `table(name: str, columns: list[Column]) -> Table`
Declares table metadata. Validates identifier syntax, prevents duplicate columns, and tracks primary key.

### `col_int(name: str, is_pk: bool = False, is_null: bool = False, is_auto: bool = False) -> Column`
Declares integer column.

### `col_str(name: str, is_null: bool = False, is_unique: bool = False) -> Column`
Declares text column.

### `col_float(name: str, is_null: bool = False) -> Column`
Declares floating point column.

### `col_bool(name: str, is_null: bool = False) -> Column`
Declares boolean column.

### `class Model[T]`
Pairs `Table` schema with pure `decoder: (Row) -> T` and `encoder: (T) -> dict[str, Value]`.

- `decode(row: Row) -> T`: Maps a raw `Row` to typed object `T`.
- `encode(obj: T) -> dict[str, Value]`: Serializes typed object `T` to column-value map.
- `decode_rows(rows: list[Row]) -> list[T]`: Maps list of rows to list of typed objects.

---

## Query Builder

### `select(table_name: str, columns: list[str]) -> SelectQuery`
Initializes a `SELECT` query projection.

### `select_all(table_name: str) -> SelectQuery`
Initializes a `SELECT *` query projection.

### `SelectQuery` Methods:
- `where(pred: Predicate) -> SelectQuery`: Adds initial WHERE predicate.
- `and_where(pred: Predicate) -> SelectQuery`: Adds predicate with AND logic.
- `or_where(pred: Predicate) -> SelectQuery`: Adds predicate with OR logic.
- `where_group(grp: WhereClause, logic: int = 1) -> SelectQuery`: Appends a nested parenthesized WHERE clause group (e.g. `(role = 'admin' OR role = 'moderator')`).
- `and_where_group(grp: WhereClause) -> SelectQuery`: Appends a parenthesized group with AND logic.
- `or_where_group(grp: WhereClause) -> SelectQuery`: Appends a parenthesized group with OR logic.
- `order_by(col: str, direction: int = 1) -> SelectQuery`: Adds ORDER BY clause (`ORDER_ASC` or `ORDER_DESC`).
- `limit(n: int) -> SelectQuery`: Sets nonnegative LIMIT.
- `offset(m: int) -> SelectQuery`: Sets nonnegative OFFSET.
- `inner_join(target_tbl: str, left_col: str, right_col: str) -> SelectQuery`: Appends INNER JOIN.
- `left_join(target_tbl: str, left_col: str, right_col: str) -> SelectQuery`: Appends LEFT JOIN.
- `compile(dialect: Dialect) -> CompiledQuery`: Compiles AST into dialect SQL and bound parameters.

### Predicates:
- `eq(col: str, val: Value) -> Predicate`: `col = ?`
- `neq(col: str, val: Value) -> Predicate`: `col != ?`
- `gt(col: str, val: Value) -> Predicate`: `col > ?`
- `gte(col: str, val: Value) -> Predicate`: `col >= ?`
- `lt(col: str, val: Value) -> Predicate`: `col < ?`
- `lte(col: str, val: Value) -> Predicate`: `col <= ?`
- `like(col: str, pattern: str) -> Predicate`: `col LIKE ?`
- `is_null(col: str) -> Predicate`: `col IS NULL`
- `is_not_null(col: str) -> Predicate`: `col IS NOT NULL`
- `is_in(col: str, vals: list[Value]) -> Predicate`: `col IN (?, ?)` or `1 = 0` if empty list.

---

## Transactions

### `begin_transaction(s: Session) -> Transaction`
Begins transaction on session and returns active `Transaction`.

### `class Transaction`
- `begin() -> None`: Starts transaction.
- `commit() -> None`: Commits transaction.
- `rollback() -> None`: Rolls back transaction.
- `is_active() -> bool`: Returns `True` if transaction is in `ACTIVE` state.

---

## Connection Pooling

### `create_pool(factory: () -> FeatherConn, dialect: Dialect, max_connections: int = 10, ping_on_acquire: bool = True) -> ConnectionPool`
Creates a bounded connection pool. `ping_on_acquire` configures whether idle connections are verified with a liveness probe prior to leasing.

### `create_sqlite_pool(db_path: str, max_connections: int = 10, ping_on_acquire: bool = True) -> ConnectionPool`
Creates a bounded SQLite connection pool.

### `class ConnectionPool`
- `acquire() -> Session`: Reuses a healthy idle connection (verifying liveness via `SELECT 1` ping when `ping_on_acquire = True`) or creates a fresh one via factory. Wraps connection in a Session with a monotonic `lease_id`.
- `release_connection(s: Session) -> None`: Validates lease, only rolls back if `s.in_transaction == True` (safely isolating and discarding poisoned connections if rollback fails), invalidates session (`is_released = True`), and returns clean healthy connection to idle pool. Clean (committed or non-transactional) sessions skip redundant rollbacks entirely. Raises `PoolError` on double-release or foreign session.
- `close_all(force: bool = False) -> None`: Closes all idle connections in pool and marks pool closed. If `force=False` (graceful default), in-flight leased sessions can finish and close upon release; if `force=True`, active leased connections are terminated immediately.
- `close_graceful() -> None`: Initiates graceful pool shutdown.
- `close_force() -> None`: Initiates immediate force pool shutdown.
- `in_use_count() -> int`: Number of currently borrowed active leases.
- `idle_count() -> int`: Number of available idle connections.
- Invariant guaranteed: `in_use_count() + idle_count() <= max_connections`.

---

## Migrations

### `migration(version: int, name: str, up_sqls: list[str], down_sqls: list[str]) -> Migration`
Defines a versioned migration.

### `compute_legacy_migration_checksum(m: Migration) -> str`
Computes legacy 8-character DJB2 checksum for backward compatibility with v0.2.0 migrations.

### `generate_lock_token() -> str`
Generates a cryptographically strong, collision-free RFC-4122 v4 UUID string for migration lock ownership.

### `class MigrationManager`
- `__init__(session: Session, migrations: list[Migration])`: Validates migration uniqueness (raising `MigrationError` on duplicate versions) and automatically sorts migrations in ascending version order via deterministic insertion sort.
- `verify_checksums() -> None`: Performs two-phase verification across all applied migrations. Validates legacy checksums in-memory first; only if all migrations pass is an atomic batch update executed to upgrade legacy checksums to SHA-256. If any record fails, if an applied migration in the database is missing from the migration list, or if a migration is in `FAILED` state, an explicit `MigrationError` is raised.
- `acquire_lock(locked_by: str = "", timeout_seconds: int = 300) -> str`: Atomically acquires the migration concurrency lock via a single conditional `UPDATE` on `_feather_migration_lock`. Automatically reclaims stale locks where `timeout_at < current_timestamp`. Returns a unique UUID ownership token string. Raises `MigrationError` if actively held by another runner.
- `refresh_lock(owner_token: str = "", additional_seconds: int = 300) -> bool`: Atomically extends the lease expiration of an actively held lock. Returns `True` if refreshed, or `False` if the lease was lost, expired, or reclaimed.
- `release_lock(owner_token: str = "") -> None`: Releases the migration lock with ownership verification. Raises `MigrationError` if `owner_token` does not match the active lock holder.
- `repair_migration_status(version: int, new_status: str, force_unlock: bool = False) -> None`: Administratively resolves a failed or in-flight migration record under migration lock. Validates `new_status` against `['APPLIED', 'RUNNING', 'FAILED']`. Modifying an already `APPLIED` migration requires explicit `force_unlock=True`.
- `drop_migration_record(version: int, force_unlock: bool = False) -> None`: Removes a migration record from `_feather_migrations` under migration lock. Dropping an already `APPLIED` migration requires explicit `force_unlock=True`.
- `preview_up() -> list[str]`: Completely side-effect-free dry run returning all pending UP SQL statements without acquiring locks or creating/modifying tables. Propagates fatal database errors.
- `preview_down(target_version: int) -> list[str]`: Completely side-effect-free dry run returning all rollback SQL statements to reach `target_version` without database mutation. Propagates fatal database errors.
- `plan_up() -> list[str]`: Alias for `preview_up()`.
- `plan_down(target_version: int) -> list[str]`: Alias for `preview_down(target_version)`.
- `migrate_up() -> int`: Executes pending migrations inside transactions (or non-transactional DDL state-tracked mode on MySQL with `RUNNING`, `APPLIED`, `FAILED` transitions) with active step-level lease fencing after acquiring concurrency lock and verifying checksums.
- `migrate_down(target_version: int) -> int`: Validates checksums with `verify_checksums()` prior to rollback, then rolls back migrations in strictly descending version order (`3 -> 2 -> 1`) down to target under concurrency lock with step-level lease fencing.
- `get_applied_versions() -> list[int]`: Returns list of successfully applied version numbers (`status == 'APPLIED'`). Raises `MigrationError` if any migration is recorded in `FAILED` or `RUNNING` status.
- `get_pending_migrations() -> list[Migration]`: Returns list of unapplied migrations.
- `status() -> list[str]`: Returns status report strings for each migration.

---

## Relationships

### `has_one(name: str, source_table: str, target_table: str, foreign_key: str, primary_key: str) -> Relation`
Defines one-to-one foreign key relation.

### `has_many(name: str, parent_table: str, child_table: str, foreign_key: str, primary_key: str) -> Relation`
Defines one-to-many foreign key relation.

### `belongs_to(name: str, child_table: str, parent_table: str, foreign_key: str, primary_key: str) -> Relation`
Defines belongs-to foreign key relation.

### `class RelationLoader`
- `load_has_many[C](child_model: Model[C], fk_column: str, parent_pk_val: Value) -> list[C]`
- `load_has_many_batch[C](child_model: Model[C], fk_column: str, parent_pk_vals: list[Value]) -> list[C]`: Eagerly loads all matching children for multiple parent IDs in a single `WHERE fk IN (?, ?, ...)` query with chunking (`BATCH_CHUNK_SIZE = 500`), eliminating N+1 queries.
- `load_relation_has_many_batch[C](rel: Relation, child_model: Model[C], parent_pk_vals: list[Value]) -> list[C]`: Relation wrapper for batched has-many loading.
- `load_has_many_grouped[C](child_model: Model[C], fk_column: str, parent_pk_vals: list[Value], get_fk: (C) -> int) -> dict[int, list[C]]`: Eagerly loads and groups matching children into a dictionary keyed by parent integer foreign key (`parent_pk -> list[Child]`).
- `load_relation_has_many_grouped[C](rel: Relation, child_model: Model[C], parent_pk_vals: list[Value], get_fk: (C) -> int) -> dict[int, list[C]]`: Relation wrapper for integer grouped loading.
- `load_has_many_grouped_str[C](child_model: Model[C], fk_column: str, parent_pk_vals: list[Value], get_fk_str: (C) -> str) -> dict[str, list[C]]`: Eagerly loads and groups matching children into a dictionary keyed by parent string / UUID foreign key (`parent_pk_str -> list[Child]`).
- `load_relation_has_many_grouped_str[C](rel: Relation, child_model: Model[C], parent_pk_vals: list[Value], get_fk_str: (C) -> str) -> dict[str, list[C]]`: Relation wrapper for string / UUID grouped loading.
- `load_belongs_to[P](parent_model: Model[P], pk_column: str, fk_val: Value) -> P | None`
- `load_belongs_to_batch[P](parent_model: Model[P], pk_column: str, fk_vals: list[Value]) -> list[P]`: Eagerly loads all matching parent entities for multiple foreign key values in a single `WHERE pk IN (?, ?, ...)` query.
- `load_relation_belongs_to_batch[P](rel: Relation, parent_model: Model[P], fk_vals: list[Value]) -> list[P]`: Relation wrapper for batched belongs-to loading.
- `load_has_one[C](child_model: Model[C], fk_column: str, parent_pk_val: Value) -> C | None`
- `load_many_to_many[T](target_model: Model[T], through_table: str, source_fk: str, target_fk: str, source_pk_val: Value, target_pk_col: str = "id") -> list[T]`
- `load_relation_many_to_many[T](rel: Relation, target_model: Model[T], source_pk_val: Value) -> list[T]`

### `many_to_many(name: str, source_table: str, target_table: str, through_table: str, source_fk: str, target_fk: str, target_pk: str = "id") -> Relation`
Defines a many-to-many relationship through a pivot/join table.

---

## Batch & Advanced CRUD Operations

### `session.insert_many(tbl: Table, rows: list[dict[str, Value]]) -> int`
Batch-inserts multiple records into `tbl` in a single SQL statement using multi-row `VALUES (?, ?), (?, ?)...`.

### `session.upsert(tbl: Table, values: dict[str, Value], conflict_target: list[str], update_values: dict[str, Value]) -> int`
Executes an upsert operation. Emits `ON CONFLICT (...) DO UPDATE SET ...` / `DO NOTHING` on SQLite & PostgreSQL, and `ON DUPLICATE KEY UPDATE` on MySQL.

### Composite Primary Key CRUD
- `session.update_by_pk(tbl: Table, pk_values: dict[str, Value], values: dict[str, Value]) -> int`
- `session.delete_by_pk(tbl: Table, pk_values: dict[str, Value]) -> int`
- `session.find_by_pk[T](model: Model[T], pk_values: dict[str, Value]) -> T | None`

---

## SQL Aggregates & Grouping

- `count_all() -> str`: Emits `COUNT(*)`.
- `count_col(col: str) -> str`: Emits `COUNT("col")`.
- `sum_col(col: str) -> str`: Emits `SUM("col")`.
- `avg_col(col: str) -> str`: Emits `AVG("col")`.
- `min_col(col: str) -> str`: Emits `MIN("col")`.
- `max_col(col: str) -> str`: Emits `MAX("col")`.
- `SelectQuery.group_by(cols: list[str]) -> SelectQuery`: Emits `GROUP BY "col1", "col2"`.
- `SelectQuery.having(pred: Predicate) -> SelectQuery`: Emits parameterized `HAVING` clause.

---

## Lossless Type Codecs

- `decimal(str_val: str) -> Decimal`: Lossless fixed-point / monetary value.
- `datetime(year, month, day, hour, minute, second) -> DateTime`: ISO-8601 UTC date-time.
- `parse_iso_datetime(iso_str: str) -> DateTime`: Parses ISO-8601 timestamp string (`YYYY-MM-DDTHH:MM:SSZ`).
- `uuid_parse(uuid_str: str) -> UUID`: Validates and parses 36-char canonical hex UUID string.
- Row getters: `row_get_decimal`, `row_get_opt_decimal`, `row_get_datetime`, `row_get_opt_datetime`, `row_get_uuid`, `row_get_opt_uuid`.
- Value constructors: `val_decimal`, `val_datetime`, `val_uuid`.
