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

## ADR 008: Poisoned Connection Isolation and Monotonic Leases

- **Context:** Sockets severed mid-transaction or failed commits can corrupt connection state. Attempting subsequent rollbacks or re-leasing damaged connections causes cascading failures.
- **Decision:** Track `Session.is_poisoned` and `Session.lease_id`. Discard severed connections immediately upon release without running redundant rollbacks. Enforce capacity invariants (`total = active + idle <= max`). Block queries on poisoned sessions while permitting rollback cleanup.
- **Consequences:** Bulletproof connection pool stability, zero capacity leaks, and immediate quarantine of broken sockets.

## ADR 009: Two-Phase Tamper-Resistant Migration Checksums

- **Context:** Migration files must be guarded against post-deployment tampering. Migrating from 8-character hashes to SHA-256 without breaking existing databases requires smooth upgrades.
- **Decision:** Use standard SHA-256 checksums across UP and DOWN SQLs. Perform a two-phase check: validate legacy checksums in-memory, and only when all migrations verify, atomically upgrade database records to SHA-256. Reject duplicate or unknown applied versions.
- **Consequences:** Uncompromising cryptographic integrity with backwards compatibility.

## ADR 010: Grouped Batch Relations and Large IN Chunking

- **Context:** Resolving 1:N relations across many parents via individual queries causes N+1 performance degradation. Exceeding SQLite's 999 parameter limit in raw `IN (...)` queries crashes drivers.
- **Decision:** Implement `load_has_many_grouped` returning `dict[int, list[Child]]` (and `load_has_many_grouped_str` for string/UUID keys) in a single batched query, automatically chunked into 500-element slices.
- **Consequences:** Eliminates N+1 queries cleanly, supports UUID/string PKs, and guarantees zero parameter limit violations.

## ADR 011: Truly Atomic Migration Concurrency Locking & Non-Transactional DDL State Tracking

- **Context:** High-concurrency or multi-instance deployments risk simultaneous migration execution. Multi-step SELECT then INSERT/UPDATE patterns create race conditions. Furthermore, MySQL executes DDL with implicit commits, making traditional transaction rollbacks impossible.
- **Decision:**
  1. Pre-initialize a singleton lock row (`id = 1`) via dialect-specific SQL (`INSERT OR IGNORE`, `ON CONFLICT DO NOTHING`, `ON DUPLICATE KEY UPDATE`).
  2. Perform lock acquisition in a single atomic database statement:
     `UPDATE _feather_migration_lock SET is_locked = 1, locked_at = ?, locked_by = ?, timeout_at = ? WHERE id = 1 AND (is_locked = 0 OR timeout_at < ?)`.
  3. Enforce lock ownership matching on release (`release_lock(owner_token)`).
  4. Automatically recover stale locks when `timeout_at < current_timestamp`.
  5. For non-transactional DDL engines (MySQL), track execution states (`RUNNING`, `APPLIED`, `FAILED`) in `_feather_migrations`, preventing corrupted retries of partially applied DDL.
  6. Provide zero-side-effect dry runs (`preview_up`, `preview_down`).
- **Consequences:** 100% database-engine atomicity, cross-runner ownership protection, resilient stale-lock recovery, and schema protection against MySQL non-transactional DDL failures.

## ADR 012: Cryptographic Token Uniqueness, Active Lease Fencing, and Metadata Table Self-Upgrade

- **Context:** User-provided worker names (`locked_by`) or sequential IDs are not safe lock ownership tokens. Long-running migrations run the risk of lease expiration mid-execution, enabling a competing process to steal the lock and execute concurrent DDL. Additionally, upgrading Feather on existing databases fails if metadata tables (`_feather_migration_lock`, `_feather_migrations`) lack newly introduced columns.
- **Decision:**
  1. Decouple descriptive `locked_by` labels from secret ownership tokens (`owner_token`). Generate unguessable RFC-4122 v4 UUIDs using `nox.crypto.sha256` seeded by `nox.random.randint` and `nox.time.now()`.
  2. Implement active step-level lease fencing (`refresh_lock()`) before every migration execution step; immediately abort if the lease was lost or reclaimed.
  3. Integrate server-managed advisory locks (`pg_try_advisory_lock`, `GET_LOCK`) for PostgreSQL and MySQL.
  4. Implement `_ensure_metadata_columns()` in `init_table()` using zero-cost `LIMIT 0` probes to automatically upgrade legacy metadata tables via non-destructive `ALTER TABLE`.
  5. Refine dry-run error handling with `_is_table_not_found_error()` to ensure fatal connection drops or permission rejections are properly propagated.
  6. Provide administrative recovery methods (`repair_migration_status`, `drop_migration_record`) for manual operator intervention on non-transactional DDL states.
- **Consequences:** Eliminates lease hijacking, prevents split-brain execution on slow migrations, ensures seamless backward compatibility across versions, and exposes explicit administrative recovery tools.
