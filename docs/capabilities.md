# Nox Capabilities and Stdlib Evidence Matrix (docs/capabilities.md)

## 1. Compiler and Runtime Environment

- **Compiler Version:** `noxc 2.0.27`
- **Architecture:** Hosted (macOS / Linux / Windows)
- **Standard Library Path:** `~/.nox-lang/lib/nox/stdlib/nox/`
- **Runtime Object:** `noxrt.o` with POSIX & C runtime shims

## 2. Verified Language Features & Compiler Invariants

| Feature | Status | Evidence / Notes |
| ------- | ------ | ---------------- |
| Generic Classes `class Box[T]` | Supported | Verified via test: `Box[T]` instantiates properly with `Box[str]`. |
| Generic Methods `def method[T](self, x: T) -> T` | Supported | Verified: callable on class instances with explicit type arg `inst.method[T](arg)`. |
| Free Generic Functions `def fn[T](x: T)` | Limited | Calling `fn[T](x)` causes `unknown generic constructor: fn`. Generic parameters must reside on classes or class methods. |
| Union Types `T \| None` | Supported | Used in `str \| None`, `int \| None`. Type narrowing with `if var is None:` on local variables functions properly. |
| Function Types `(A, B) -> C` | Supported | First-class function values can be stored in class fields and called. Used for decoders and callbacks. |
| `protocol DbConnection` | Supported | Structural subtyping (duck typing) without explicit `implements`. Includes `close()`, `execute(sql)`, `query(sql)`, `prepare(sql)`. |
| Exceptions `class MyError(Exception)` | Supported | Full `try...except Exception as e:` and `raise MyError(msg)` supported. |
| Dictionary Operations `dict[str, Value]` | Supported | Literal `{...}`, `.contains(k)`, `d[k] = v`, `d.keys() -> list[str]` supported. |
| List Operations `list[T]` | Supported | `len(l)`, `.append(v)`, `l[i]`, empty list initialization `l: list[T] = []` when type-annotated. |
| Class Method Naming `release()` | Caution | Nox ARC codegen generates `@<mod>_<class>_release` as the destructor symbol. User methods named `release` collide with this symbol during LLVM codegen. Use `release_connection` or alternative names. |
| Formatter `noxc fmt` | Limitation | `noxc fmt` in 2.0.27 strips generic class headers (e.g. `class Model[T]:` -> `class Model:`) and `pass` in empty classes. Do not use automated in-place formatting on files containing generics. |

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
