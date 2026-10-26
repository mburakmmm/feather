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

### `create_sqlite_pool(db_path: str, max_connections: int = 10) -> ConnectionPool`
Creates a bounded SQLite connection pool.

### `class ConnectionPool`
- `acquire() -> Session`: Borrows a connection and wraps in a Session.
- `release_connection(s: Session) -> None`: Returns borrowed connection to pool.
- `close_all() -> None`: Closes all idle connections in pool.
- `in_use_count() -> int`: Number of currently borrowed connections.
- `idle_count() -> int`: Number of available idle connections.

---

## Migrations

### `migration(version: int, name: str, up_sqls: list[str], down_sqls: list[str]) -> Migration`
Defines a versioned migration.

### `class MigrationManager`
- `migrate_up() -> int`: Executes pending migrations inside transactions.
- `migrate_down(target_version: int) -> int`: Rolls back migrations down to target.
- `get_applied_versions() -> list[int]`: Returns list of applied version numbers.
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
- `load_belongs_to[P](parent_model: Model[P], pk_column: str, fk_val: Value) -> P | None`
- `load_has_one[C](child_model: Model[C], fk_column: str, parent_pk_val: Value) -> C | None`
