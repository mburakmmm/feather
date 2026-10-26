# Feather 🪶

**The lightweight, type-safe SQL data mapper and query toolkit for Nox.**

[![Nox Version](https://img.shields.io/badge/noxc-2.0.27-blue.svg)](https://noxlang.com)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Tests](https://img.shields.io/badge/tests-29%2F29%20passed-brightgreen.svg)]()

Feather makes everyday database work in the [Nox programming language](https://github.com/mburakmmm/nox-lang) intuitive, predictable, and safe. Think **Peewee / SQLModel in spirit**: explicit typed model mapping, parameterized CRUD, query construction, transaction management, connection pooling, and schema migrations — built directly on Nox's standard library with zero external dependencies.

---

## Features

- 🛡️ **SQL-First & Safe by Construction:** All values are bound via prepared statement parameters. SQL injection is prevented by design, with strict identifier verification on tables, columns, and aggregates.
- 🎯 **Explicit Static Typing:** Zero runtime metaclasses or dynamic class mutation. Pure typed row decoders and encoders.
- 🔢 **Lossless Type Codecs:** String-backed `Decimal` for precision financial math, ISO-8601 UTC `DateTime`, and RFC-4122 `UUID` validation.
- 🔍 **Fluent Query Builder & Parenthesized Grouping:** Construct complex `SELECT`, `WHERE` (`=`, `!=`, `>`, `<`, `LIKE`, `IN`, `IS NULL`), `where_group` parentheses expressions (`(A OR B) AND C`), `ORDER BY`, `LIMIT`, `OFFSET`, `INNER JOIN`, `LEFT JOIN`, `GROUP BY`, and `HAVING` queries through a type-safe AST.
- 📊 **SQL Aggregates:** `count_all()`, `sum_col()`, `avg_col()`, `min_col()`, `max_col()` with inner column identifier sanitization.
- ⚡ **Batch Insertion (`insert_many`):** Multi-row `VALUES (?, ?), (?, ?)...` generator with schema uniformity verification, preventing accidental NULL insertions.
- 🔀 **Upsert Operations:** `session.upsert` with `ON CONFLICT (...) DO UPDATE` / `DO NOTHING` (SQLite/PostgreSQL) and `ON DUPLICATE KEY UPDATE` (MySQL).
- 🔑 **Composite Primary Keys:** Full multi-column PK support with table-level constraint DDL, `find_by_pk`, `update_by_pk`, and `delete_by_pk` (with composite auto-increment safety).
- 🔄 **Multi-Dialect Support:** Portable queries across **SQLite**, **PostgreSQL** (`$1..$N` placeholders, `RETURNING`), and **MySQL** (backtick identifier quoting, `INT AUTO_INCREMENT PRIMARY KEY` migration DDL).
- 💼 **Transaction Unit of Work:** Explicit state-tracked transactions (`IDLE`, `ACTIVE`, `COMMITTED`, `ROLLED_BACK`, `FAILED`) with `s.in_transaction`, `s.is_poisoned` guard blocking operations on compromised sessions, zero-overhead disposal of failed sockets, and tokenized raw SQL transaction detection.
- 🏊 **Leased Connection Pooling:** Bounded connection pool (`ConnectionPool`) with monotonic lease tracking, poison connection isolation, configurable liveness check (`ping_on_acquire`), graceful vs force shutdown (`close_graceful()`, `close_force()`), and capacity invariant enforcement (`total = active + idle <= max`).
- 📦 **Schema Migrations with Concurrency Locks & Dry Runs:** Versioned migrations with `_feather_migration_lock` concurrency control, dry-run previews (`plan_up`, `plan_down`), dialect-aware DDL policy (accounting for MySQL implicit commit), standard `nox.crypto.sha256` tamper protection, deterministic auto-sorting of out-of-order migrations, duplicate version rejection, detection of missing/unknown applied versions, and pre-rollback validation in `migrate_down()`.
- 🔗 **Relationship Loaders & Batched Eager Loading:** Explicit `has_one`, `has_many`, `belongs_to`, and `many_to_many` loaders with `load_has_many_batch`, `load_belongs_to_batch`, `load_has_many_grouped` (`dict[int, list[C]]`), and `load_has_many_grouped_str` (`dict[str, list[C]]` for string & UUID keys) with parameter chunking eliminating N+1 queries.
- 🖥️ **CLI Migration & Verification Tool:** Run `feather demo`, `feather migrate status`, `feather migrate up`, and `feather migrate down`.
- ⚡ **Zero External Dependencies:** Built 100% on Nox standard library modules (`nox.db`, `nox.sqlite`, `nox.postgres`, `nox.mysql`, `nox.strings`, `nox.crypto`, `nox.test`, `nox.os`).

---

## Comparison: `nox.orm` vs `Feather`

| Feature | `nox.orm` (stdlib) | `Feather` |
| ------- | ------------------ | --------- |
| Object Mapping | Untyped `Row` only | Generic typed `Model[T]` with structured encode/decode |
| Query Construction | Manual SQL strings (`where_sql`) | Fluent AST builder (`eq`, `gt`, `in`, `and_`, `or_`, `join`) |
| Placeholder Handling | Hardcoded `?` only (fails on PostgreSQL) | Dialect-aware (`?` for SQLite/MySQL, `$1..$N` for PostgreSQL) |
| Transactions | None (manual SQL) | State-machine `Transaction` with automatic error rollback |
| Bulk Mutation Safety | Allows unfiltered deletes | Unfiltered `UPDATE` / `DELETE` raises `QueryBuildError` |
| Migrations | None | `MigrationManager` with versioning, rollback, and status |
| Connection Pooling | None | Bounded `ConnectionPool` with active/idle tracking |
| Foreign Key Relations | None | Explicit `RelationLoader` (`has_one`, `has_many`, `belongs_to`) |

---

## Installation

Add `feather` to your project's `nox.json`:

```json
{
  "name": "my_app",
  "version": "0.1.0",
  "entry": "main.nox",
  "requires": [
    "feather"
  ]
}
```

Or install locally/globally via `noxc`:

```bash
noxc add feather
```

---

## Quickstart

```nox
import feather
from feather import Session, Table, Column, Value, val_int, val_str, val_bool, col_int, col_str, col_bool, table, select_all, SelectQuery, eq, gt, open_sqlite_memory, row_get_int, row_get_str, row_get_bool
import mapper
from mapper import Model
import nox.db
from nox.db import Row

# 1. Define your typed entity
class User:
    id: int
    name: str
    email: str
    is_active: bool

    def __init__(self: User, id: int, name: str, email: str, is_active: bool) -> None:
        self.id = id
        self.name = name
        self.email = email
        self.is_active = is_active

# 2. Pure decoder and encoder functions
def decode_user(r: Row) -> User:
    return User(
        row_get_int(r, 0),
        row_get_str(r, 1),
        row_get_str(r, 2),
        row_get_bool(r, 3)
    )

def encode_user(u: User) -> dict[str, Value]:
    d: dict[str, Value] = {}
    d["id"] = val_int(u.id)
    d["name"] = val_str(u.name)
    d["email"] = val_str(u.email)
    d["is_active"] = val_bool(u.is_active)
    return d

def run() -> None:
    # 3. Open in-memory SQLite session
    s: Session = open_sqlite_memory()

    # 4. Declare schema metadata and bind Model
    user_tbl: Table = table("users", [
        col_int("id", True, False, True),
        col_str("name"),
        col_str("email"),
        col_bool("is_active")
    ])
    user_model: Model[User] = Model[User](user_tbl, decode_user, encode_user)
    s.create_table(user_tbl)

    # 5. Insert records
    user_id: int = s.insert(user_tbl, {
        "name": val_str("Alice Smith"),
        "email": val_str("alice@example.com"),
        "is_active": val_bool(True)
    })
    print("Created User ID: " + str(user_id))

    # 6. Query with filters
    q: SelectQuery = select_all("users").where(eq("is_active", val_bool(True)))
    users: list[User] = s.execute_query[User](user_model, q)
    print("Active users found: " + str(len(users)))

    # 7. Update by ID
    s.update_by_id(user_tbl, val_int(user_id), {"name": val_str("Alice Jones")})

    # 8. Clean teardown
    s.close()

run()
```

---

## Transaction Management

Feather provides strict, state-machine tracked transactions:

```nox
import feather
from feather import Session, Transaction, begin_transaction, val_int

s: Session = feather.connect_sqlite("app.db")
tx: Transaction = begin_transaction(s)
try:
    s.update_by_id(accounts_tbl, val_int(1), {"balance": val_int(900)})
    s.update_by_id(accounts_tbl, val_int(2), {"balance": val_int(600)})
    tx.commit()
    print("Transferred funds successfully!")
except Exception as e:
    tx.rollback()
    print("Transfer failed and rolled back: " + e.message)
finally:
    s.close()
```

---

## Connection Pooling

Reuse database handles efficiently using `ConnectionPool`:

```nox
import feather
from feather import ConnectionPool, Session, create_sqlite_pool

# Create pool with up to 10 connections
pool: ConnectionPool = create_sqlite_pool("app.db", 10)

# Acquire session
s: Session = pool.acquire()
try:
    s.execute_raw("INSERT INTO logs VALUES (?)", [feather.val_str("Ping")])
finally:
    # Release connection back to pool
    pool.release_connection(s)

# Close all connections on application shutdown
pool.close_all()
```

---

## Schema Migrations

Track and execute database schema changes safely:

```nox
import feather
from feather import Session, Migration, MigrationManager, migration

s: Session = feather.connect_sqlite("app.db")

m1: Migration = migration(
    1,
    "create_users",
    ["CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY, name TEXT)"],
    ["DROP TABLE IF EXISTS users"]
)

m2: Migration = migration(
    2,
    "add_email",
    ["ALTER TABLE users ADD COLUMN email TEXT"],
    ["DROP TABLE IF EXISTS users"]
)

mgr: MigrationManager = MigrationManager(s, [m1, m2])

# Apply all pending migrations in isolated transactions
applied: int = mgr.migrate_up()
print("Applied " + str(applied) + " migrations.")

# Check status
status_lines: list[str] = mgr.status()
```

---

## Relationships and Eager Loading

Declare relationships explicitly without the risk of hidden N+1 performance degradation:

```nox
import feather
from feather import Relation, RelationLoader, has_many, belongs_to, val_int

# Define metadata
author_books_rel: Relation = has_many("books", "authors", "books", "author_id", "id")

# Explicit loading
loader: RelationLoader = RelationLoader(session)
books: list[Book] = loader.load_has_many[Book](book_model, "author_id", val_int(author.id))
```

---

## Running Tests

Run the full Feather test suite using the standard Nox test runner:

```bash
noxc test
```

All 29 test suites run against SQLite in-memory fixtures without requiring external database services.

---

## Architecture and Documentation

- [Architecture Overview](docs/architecture.md)
- [API Reference](docs/api.md)
- [Model Mapping Guide](docs/model-mapping.md)
- [Query Builder Guide](docs/queries.md)
- [Transactions Guide](docs/transactions.md)
- [Dialects Reference](docs/dialects.md)
- [Connection Pooling](docs/pooling.md)
- [Schema Migrations](docs/migrations.md)
- [Relationships Guide](docs/relations.md)
- [Security Invariants](docs/security.md)
- [Nox Capabilities Matrix](docs/capabilities.md)
- [Architectural Decisions](docs/decisions.md)
- [Compatibility Matrix](docs/compatibility.md)
- [Performance & Memory](docs/performance.md)
- [Implementation Roadmap](docs/roadmap.md)
- [Migration Guide](docs/migration-guide.md)

---

## License

MIT License. Copyright (c) 2026 Melih Burak Memis. See [LICENSE](LICENSE) for details.
