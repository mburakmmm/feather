# Architectural Decisions Record (docs/decisions.md)

## ADR 001: Explicit Model Metadata and Decoders over Metaclasses

- **Context:** Nox is a statically-typed language without Python-style dynamic metaclasses, class decorators that alter class structure at runtime, or compile-time reflection macros.
- **Decision:** Models are defined with explicit `Table` metadata and pure decoder/encoder functions. A generic `Model[T]` pairs the table metadata with typed `(Row) -> T` and `(T) -> dict[str, Value]` functions.
- **Consequences:** Eliminates runtime magic, ensures 100% static type safety, and avoids compiler/runtime crashes.

## ADR 002: Structured Value Union and Zero-Based Parameter Binding

- **Context:** Nox database drivers expect zero-based parameter bindings for `Statement.bind_*`. Concatenating values into SQL strings introduces severe injection risks.
- **Decision:** All query values are wrapped in a typed `Value` union (`val_int`, `val_str`, `val_float`, `val_bool`, `val_null`). The query compiler generates parameter lists in exact lockstep with placeholders.
- **Consequences:** Safe by construction. Zero SQL injection vulnerability for values.

## ADR 003: Dialect Abstraction Layer

- **Context:** SQLite and MySQL use `?` placeholders, while PostgreSQL requires numbered placeholders (`$1, $2, ...`). SQLite/PostgreSQL use `"` for identifiers, whereas MySQL uses `` ` ``. Key generation differs across engines.
- **Decision:** Introduce a `Dialect` class with dialect-specific methods (`quote_identifier`, `placeholder`, `column_type_sql`, `create_table_sql`, `limit_offset_sql`).
- **Consequences:** Queries compile cleanly to the targeted engine's dialect without leaky cross-engine syntax assumptions.

## ADR 004: State-Tracked Explicit Transactions

- **Context:** Database operations can fail mid-stream. Incomplete transactions leave connection state corrupted or tables locked.
- **Decision:** Provide an explicit `Transaction` object tracking states (`IDLE`, `ACTIVE`, `COMMITTED`, `ROLLED_BACK`, `FAILED`). Rollback is automatically triggered on unhandled exceptions in transaction scopes.
- **Consequences:** Clean lifecycle, no lingering open transactions, and strict ownership semantics.

## ADR 005: Built-in Connection Pooling

- **Context:** Creating new database connections per request is expensive.
- **Decision:** Implement a light, deterministic `ConnectionPool` with `acquire()` and `release()` semantics, tracking active and idle connections.
- **Consequences:** Reusable connections with predictable resource boundaries and zero external dependencies.

## ADR 006: Versioned Migration Manager

- **Context:** Applications need repeatable, reversible schema definitions.
- **Decision:** Provide a migration runner tracking applied versions in a metadata table (`_feather_migrations`). Support both forward (`up`) and rollback (`down`) scripts within transactions.
- **Consequences:** Clean, deterministic schema evolution.

## ADR 007: Declarative Foreign-Key Relationship Loaders

- **Context:** Full active-record graph persistence leads to unpredictable N+1 queries.
- **Decision:** Define explicit relationships (`HasOne`, `HasMany`, `BelongsTo`) and provide explicit eager loader methods that execute bounded, transparent queries.
- **Consequences:** Highly predictable query count, zero hidden lazy database fetches.
