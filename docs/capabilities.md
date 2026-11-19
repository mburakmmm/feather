# Nox Capabilities and Stdlib Evidence Matrix (docs/capabilities.md)

## 1. Compiler and Runtime Environment

- **Compiler Version:** `noxc 2.2.38` (validated against Nox 2.2.38 language, stdlib & ARC improvements; prior baselines: `2.0.27`, `2.1.186`)
- **Architecture:** Hosted (macOS / Linux / Windows) + Pure-Nox Portable Subset (WASM / Mobile AST & SQL generation)
- **Standard Library Path:** `~/.nox-lang/lib/nox/stdlib/nox/`
- **Runtime Object:** `noxrt.o` with POSIX & C runtime shims

## 2. Verified Language Features & Compiler Invariants

| Feature | Status | Evidence / Notes |
| ------- | ------ | ---------------- |
| Generic Classes `class Box[T]` | Supported | Verified via `Model[T]`. Inline generic return values (e.g. `s.insert(tbl, model.encode(w))`) compile and run cleanly on `noxc 2.2.38` (`compiler_regression_2_2_test.nox`). |
| Generic Methods `def method[T](self, x: T) -> T` | Supported | Verified on `Session` (`find_all[T]`, `stream[T]`, `stream_keyset[T]`). Explicit `[T]` calls require positional arguments. |
| Generic Functions `def fn[T](x: T)` | Supported (with direct import) | `dataclass_model[T]` and `reflect_dataclass_model[T]` compile cleanly when imported from `mapper` / `reflect_schema` and called with positional arguments. |
| Union Types `T \| None` & Narrowing | Supported | Supports local `if var is not None:`, `and`/`or` expressions (`2.1.195`), and attribute-path narrowing (`if obj.field is not None:` for pointer-backed optionals in `2.2.37`). |
| Tagged Unions (`enum`) & Scalar `match` | Supported | Verified via `enum SqlValue` AND scalar `match` (`int`, `str`, `bool` in `errors.nox`, `dialects.nox`, `mapper.nox`, `reflect_schema.nox`, `value.nox` — LLVM lowered to jump tables in `2.2.38`). |
| Context Managers (`with`) & `__del__` | Supported | Verified on `Transaction`, `PoolLease`, and compound `with cp.lease() as s, s.transaction() as tx:` (`2.2.21` same-line collision fix). |
| Iterators & Generators (`yield` / `Iterator[T]`) | Supported | Verified on `Session.stream`, `stream_rows`, and `stream_keyset` with lazy iteration and early `break`. |
| Variadic & Keyword-Only Parameters | Supported | Verified in `in_vals`, `select_cols`, `group_by_cols`, `paginate`, `run_in_tx`, `update_fields`. |
| `@dataclass`, `@derive("copy", "json")`, `nox.reflect` | Supported | Verified with custom `@table`, `@primary_key`, `@auto_increment`, `@nullable`, `@unique` decorators in `reflect_schema.nox`. |
| Exception Chaining (`raise ... from`, bare `raise`) | Supported | Verified in `errors.nox`, `types.nox`, `mapper.nox`, `migrations.nox`, `session.nox` preserving `Exception.cause`. |
| Module Visibility (`__all__`) | Supported (fns/classes only) | Opt-in `__all__` enforced on `errors`, `types`, `mapper`, `reflect_schema`, `pool`, `migrations`. Modules exporting constants/enums (`value`, `metadata`, `dialects`, `query`, `session`, `relations`) omit `__all__` because `noxc 2.2.38` restricts `__all__` to module-defined functions and classes. |
| Spread, Type Aliases & `:=` Inference | Supported | Verified `{**d1, **d2}`, `[*a, *b]`, non-generic module-level `type Alias = Target`, multi-line `from m import (...)`, and local `:=` inside functions (`2.2.20`–`2.2.22`). |
| Zero-Alloc `nox.json.pos_*` & `sqlite.execute_script` | Supported | Verified tape navigation (`pos_root`/`pos_kind`/`pos_get`/`pos_at`) in `mapper.nox` and multi-statement SQLite DDL/DML via `Session.execute_script` (`2.1.195` + `2.2.4`). |
| Pure-Nox Portability (`nox.json`, `nox.path`, `nox.crypto`) | Supported | Core modules intentionally avoid C-backed `nox.xxhash`, `nox.yyjson`, and `nox.pcre2` so compilation succeeds on WebAssembly and mobile targets. |
| Class Method Naming `release()` | Caution | Nox ARC codegen generates `@<mod>_<class>_release` as the destructor symbol. User methods named `release` collide with this symbol. Use `release_connection` or alternative names. |
| Variable Name Shadowing Imported Modules | Caution | Avoid naming local variables identically to imported module names (e.g., use `cp` instead of `pool` when `pool` is an imported module). |
| Formatter `noxc fmt` | Limitation | Do not run automated `noxc fmt` on files containing generic class headers without verifying diff. |

## 3. Database Drivers Audit

### 3.1 SQLite (`nox.sqlite`)
- **Driver:** System `libsqlite3` loaded dynamically via `std.DynLib`.
- **Placeholder Format:** `?` with 0-based parameter binding in Nox (internally mapped to 1-based `sqlite3_bind_*`).
- **Connection API:** `nox.sqlite.open(path: str) -> Connection`. Supports in-memory database with `":memory:"`.
- **Prepared Statements:** Single-use `Statement` (`bind_*`, `execute()`, `query()`, followed by `finalize`).
- **Generated Key:** `conn.last_insert_rowid()`.
- **Changes Count:** `conn.changes()`, also returned by `execute()`.
- **Transaction Model:** Explicit SQL `BEGIN`, `COMMIT`, `ROLLBACK`.

### 3.2 PostgreSQL (`nox.postgres`)
- **Driver:** System `libpq` loaded dynamically via `std.DynLib`.
- **Placeholder Format:** `$1`, `$2`, ... native parameter numbering.
- **Connection API:** `nox.postgres.open(conninfo: str) -> Connection`.
- **Prepared Statements:** Client-side parameter accumulation sent via `PQexecParams`.
- **Generated Key:** Supported via `INSERT INTO ... RETURNING <pk>`. No `last_insert_rowid()`.
- **Changes Count:** Returned by `execute()`.
- **Transaction Model:** Explicit SQL `BEGIN`, `COMMIT`, `ROLLBACK`.

### 3.3 MySQL / MariaDB (`nox.mysql`)
- **Driver:** System `libmysqlclient` / MariaDB connector loaded dynamically via `std.DynLib`.
- **Placeholder Format:** `?` placeholder format.
- **Connection API:** `nox.mysql.open_url(url: str) -> Connection` with `mysql://user:pass@host:port/db`.
- **Prepared Statements:** Client-side escaping using `mysql_real_escape_string` and query dispatch.
- **Generated Key:** `conn.last_insert_rowid()` (via `mysql_insert_id`).
- **Changes Count:** Returned by `execute()`.
- **Transaction Model:** Explicit SQL `START TRANSACTION` / `BEGIN`, `COMMIT`, `ROLLBACK`.

## 4. Stdlib Comparison: `nox.orm` vs `feather`

| Capability | `nox.orm` (stdlib) | `feather` (ORM Toolkit) |
| ---------- | ------------------ | ---------------------- |
| Object Mapping | None (raw `Row` only) | Typed `Model[T]` with structured encode/decode |
| Query Builder | String templates (`where_sql`) | Full AST query builder (`eq`, `and_`, `in`, `order_by`, `limit`, `offset`, `join`) |
| Dialect Handling | SQLite `?` only | Multi-dialect (SQLite, PostgreSQL `$1..$N`, MySQL backtick quotes) |
| Transaction Support | None (raw SQL) | State-tracked `Transaction` with automatic error rollback |
| Connection Pooling | None | `ConnectionPool` with configurable capacity and reuse |
| Schema Migrations | None | `MigrationManager` with versioning, up/down runs, tracking table |
| Relationships | None | `has_one`, `has_many`, `belongs_to` foreign key relations and eager loaders |
| Error Handling | Generic `OrmError` | Typed hierarchy (`QueryBuildError`, `ConstraintError`, `TransactionError`) |
