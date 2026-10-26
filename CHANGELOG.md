# Changelog

All notable changes to **Feather** will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
