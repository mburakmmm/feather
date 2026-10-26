# AGENTS.md — FEATHER
## The lightweight, Nox-native SQL data mapper and query toolkit

> **Binding engineering contract for Cursor, Claude Code, Codex, and all AI coding agents.**
>
> **Canonical project name:** **Feather**. This is the renamed **Sable** concept. Do not use `Sable` in new source files, package metadata, documentation, imports, examples, release names, or public APIs.
>
> **Mission:** Build a compact, predictable, secure SQL toolkit for the Nox language: ergonomic typed model mapping, parameterized CRUD, query construction, explicit transactions, and optional future migrations and relationships. Think **Peewee / SQLModel in spirit**, not a giant SQLAlchemy clone.

---

# 1. Product vision and boundaries

Feather should make everyday database work in Nox significantly easier without hiding SQL or duplicating its standard library.

**Supported database families (subject to actual driver verification):**
- SQLite
- PostgreSQL
- MySQL / MariaDB

**Core principles:**
1. **SQL-first, not SQL-hostile.** Raw SQL is a supported escape hatch, not a failure.
2. **Thin over Nox.** Build on `nox.db`, `nox.sqlite`, `nox.postgres`, `nox.mysql`, and selectively `nox.orm`.
3. **Static typing.** Respect the actual Nox type system. No fictional Python metaclasses, dynamic model injection, or runtime magic.
4. **Small public API.** A developer should understand common CRUD after reading one page.
5. **Predictable performance.** No invisible N+1 queries, implicit commits, or surprising eager loads.
6. **Security by construction.** Parameterize values; validate and quote identifiers by dialect.
7. **Cross-database honesty.** Document differences rather than pretending all engines behave identically.
8. **Framework independence.** Nyx, Aether, Aura, Nova, Noxen, and Rune must not be mandatory dependencies.
9. **Production discipline.** Transactions, resource cleanup, errors, schema compatibility, and tests are not optional.
10. **Incremental development.** MVP first; advanced migrations/relationships/pooling later.

**Not in MVP:** automatic schema inference from arbitrary classes, auto-migrations, active record magic, lazy relationships, distributed transactions, multi-database joins, implicit connection pooling, GraphQL, admin panel, async DB access, automatic data validation framework, or SQL query execution from arbitrary user-provided fragments.

---

# 2. Mandatory discovery and evidence gate

Before implementation, inspect:
- `https://github.com/mburakmmm/nox-lang`
- `https://noxlang.com`
- Current Feather repository, README, tests, existing code, `AGENTS.md`, and architectural decisions.
- Current Nox compiler version via `noxc version`.
- Nox docs/source for `nox.db`, `nox.orm`, `nox.sqlite`, `nox.postgres`, `nox.mysql`.
- Nox docs for protocols, generics, classes, reflection, nullable types, exception handling, modules, packaging, concurrency, memory management, testing.
- Actual source for the driver and statement implementations when behavior is unclear.

Create `docs/capabilities.md` and `docs/decisions.md`. For every feature record:
- exact API and source inspected;
- Nox/compiler version;
- tested driver/backend;
- supported, limited, or unsupported;
- fallback or explicit omission.

**Never design around a language feature or database primitive without compiling a minimal proof of concept.**

## 2.1 Verified Nox documentation baseline — revalidate locally

From the Nox repository's documented stdlib:

- `nox.db` defines `DbConnection`, `Row`, `Statement`.
- `DbConnection` has `execute(sql)`, `query(sql)`, `prepare(sql)`, `close()`.
- `Row` offers indexed `get_str`, `get_int`, `get_float`, `is_null`, `column_count`, `column_name`. **Columns are indexed from zero.**
- `Statement` offers `bind_int`, `bind_float`, `bind_str`, `bind_null`, then `execute()` or `query()`.
- **Statements are single-use** after execution/query; do not cache/re-execute a finished `Statement`.
- **SQLite/MySQL use `?`; PostgreSQL uses `$1`, `$2`, ...** for placeholders. Binding indexes in Nox remain zero-based for all.
- `nox.orm` already provides `Table`, `Column`, `Value`, `create_table`, `insert`, `update`, `delete`, `select`. Its documented generated SQL uses `?`, so it is **not automatically portable to PostgreSQL**.
- `nox.orm` is a micro-ORM / typed-value query builder, **not an object mapper**. Feather must justify what it adds.
- SQLite and MySQL expose `last_insert_rowid`; PostgreSQL uses `INSERT ... RETURNING`.
- SQLite connections are not safe to share between tasks running on different workers; PostgreSQL documentation likewise cautions against cross-worker sharing.
- SQLite loads system `libsqlite3`; PostgreSQL loads `libpq`; MySQL/MariaDB loads a compatible client library at runtime.
- Nox's documented MySQL driver uses client-side escaped substitution rather than server-side named prepared statements; do not claim all three drivers offer identical server-side prepared statement semantics.
- The documented `Row` representation is text-based with typed getters, so precision and SQL type fidelity must be examined.

**Not established by these docs:** driver-native transaction/savepoint APIs, connection pooling, async database operations, named prepared statements, streaming cursors, typed date/time/decimal/UUID/blob bindings, database schema introspection, generic runtime class construction, lazy relationship proxies, or robust cross-driver error codes. Treat these as **unverified**.

## 2.2 Important distinction: existing `nox.orm`

Before creating a feature, compare it with `nox.orm`:
- If the stdlib already does it correctly and portably, reuse it.
- If it only works with `?`, use a dialect-aware Feather query compiler rather than assuming PostgreSQL compatibility.
- Feather's real added value should be **typed mapping + coherent query API + dialect correctness + explicit transactions + consistent errors**, not merely renamed CRUD wrappers.
- Do not copy the stdlib implementation wholesale.

---

# 3. Non-negotiable coding rules

1. Write idiomatic **Nox**, not Python, Go, Rust, or pseudocode presented as Nox.
2. Every function/field follows Nox's mandatory static typing.
3. Do not invent decorators (`@model`, `@column`), metaclasses, annotations, reflection capabilities, or macros.
4. If Nox reflection supports a capability, verify it with a compiling test before relying on it.
5. Do not modify the Nox compiler, Zig runtime, stdlib, or other repositories without explicit user permission.
6. Never concatenate untrusted SQL **values** into query text.
7. Never treat bound parameters as protection for SQL **identifiers** (table names, columns, ordering direction).
8. No implicit transaction commit/rollback on unrelated connections.
9. No hidden global database connection.
10. No connection sharing across tasks/workers unless the driver explicitly guarantees it.
11. No silently dropped rows, ignored query filters, ignored validation failures, or fake migration success.
12. No fabricated PostgreSQL/MySQL support based only on SQLite tests.
13. No silent precision loss for money, large integers, dates, binary data, or nullable fields.
14. No implicit data deletion from schema synchronization.
15. No network database requirement for core unit tests; use SQLite memory fixtures.
16. No claims of async/pooling/streaming until actually supported and tested.
17. No unsafe default logging of SQL values, passwords, or connection strings.
18. No mandatory dependencies on Nyx/Aether/Aura or any UI framework.
19. No fake API compatibility with SQLAlchemy/SQLModel/Peewee.
20. Every public feature requires tests, docs, and compatibility notes.

---

# 4. Architecture

Recommended layered design:

```text
                  Application / Nyx / Aether (optional)
                                 |
                          Feather public API
                                 |
              +------------------+------------------+
              |                  |                  |
           Models              Query              Session
       metadata + codecs     AST + builder      unit of work
              |                  |                  |
              +------------- Dialect ---------------+
                            SQL compiler
                                 |
                           Driver adapter
                                 |
             nox.db / sqlite / postgres / mysql
```

**Required direction:**
- `core` contains portable value/error/schema abstractions.
- `dialect` handles SQL syntax differences and placeholder generation.
- `query` constructs a typed/validated representation, not unsafe concatenated fragments.
- `mapper` maps rows to explicitly defined Nox model types.
- `session` coordinates connections and transactions.
- `drivers` adapt existing Nox driver behavior.
- `migration` and `relations` are optional later modules.
- No circular imports.
- The low-level SQL adapter must never import application/framework code.
- Raw SQL execution remains available through explicit, clearly named methods.

## 4.1 Suggested repository layout (not guaranteed Nox import paths)

```text
feather/
  AGENTS.md
  README.md
  LICENSE
  CHANGELOG.md
  CONTRIBUTING.md
  nox.json
  nox.lock                       # when dependencies exist
  docs/
    architecture.md
    capabilities.md
    decisions.md
    api.md
    model-mapping.md
    queries.md
    transactions.md
    dialects.md
    security.md
    migrations.md
    compatibility.md
    performance.md
    roadmap.md
    migration-guide.md
  feather/                       # adjust after verifying Nox package imports
    core.nox
    errors.nox
    values.nox
    metadata.nox
    fields.nox
    mapper.nox
    query.nox
    expressions.nox
    compiler.nox
    dialect.nox
    session.nox
    transaction.nox
    drivers.nox
    sqlite.nox
    postgres.nox
    mysql.nox
    migrations.nox              # future only
    relations.nox               # future only
  tests/
    model_test.nox
    query_test.nox
    dialect_test.nox
    sqlite_test.nox
    transaction_test.nox
    security_test.nox
    regression_test.nox
    integration/
  examples/
    quickstart.nox
    crud.nox
    query_filters.nox
    transactions.nox
    raw_sql.nox
    postgres.nox
    mysql.nox
  benchmarks/
  .github/workflows/
```

Do not generate empty files or unimplemented public methods just to match this diagram.

---

# 5. Model system: explicit and type-safe

## 5.1 Fundamental approach

Nox's documented `nox.orm` does **not** provide automatic class-to-table mapping. Feather must select an approach supported by the real language:

**Preferred MVP:** explicit model metadata + explicit row-to-model conversion, with reusable helpers.

Example **conceptual pseudocode — NOT VERIFIED NOX**:

```text
User { id: int, name: str, email: str }
users_table = table("users", columns(...))
decode_user(row) -> User
encode_user(user) -> parameter values
```

If reflection can safely discover fields, consider optional automation later, with compile/test evidence. Never require magical annotations to make the core work.

## 5.2 Model metadata

Define:
- database table name;
- ordered column definitions;
- primary key(s), including composite keys if supported;
- SQL type and Nox value conversion;
- nullability;
- optional default (client-side vs server-side distinguished);
- uniqueness/index metadata (only where used);
- generated/identity column behavior;
- optional explicit mapping between Nox field and SQL column names.

**Requirements:**
- Validate duplicate column names, missing primary key, reserved words, illegal identifiers.
- Preserve declared column order.
- Separate model type identity from database table identity.
- Do not rely on dictionary iteration order unless Nox guarantees it.
- Reject ambiguous mappings at definition time.
- Model metadata should be immutable once registered, where Nox permits.
- No runtime SQL execution during model declaration.

## 5.3 Supported types

Initial, tested types:
- `int`
- `float`
- `str`
- `bool` with dialect-aware representation
- SQL `NULL` via explicit nullability/option representation

Advanced types only with verified drivers/codecs:
- timestamps/dates/times and timezones
- decimal/money (never silently round through float)
- UUID
- JSON columns
- blobs/binary
- arrays (PostgreSQL-specific)
- enum and custom user types

**Critical:** distinguish absent field, SQL NULL, default value, and empty string. A missing field is not automatically NULL. Do not infer nullable from a default alone.

## 5.4 Mapping correctness

- Match result columns deterministically; prefer explicit select projection and ordered indices.
- If column aliases or reordered projections are used, validate mapping by column name where safe.
- Handle missing columns, duplicate names, incompatible types, NULL violations, numeric conversion failures.
- Return a clear mapping error instead of partially initialized objects.
- Do not silently cast invalid strings to zero or truncate floats to ints.
- Prevent integer overflow and unexpected float NaN/Inf behavior.
- No implicit lazy database fetches during field access.
- Document object identity semantics: detached value vs tracked entity.
- Do not invent an identity map for MVP.

---

# 6. SQL value representation and binding

Feather must use prepared/bound parameters for all data values.

- Reuse `nox.orm.Value` where it is suitable, or define a small explicit value union only when justified.
- Verify actual Nox sum types/generics before implementing.
- SQL NULL uses `bind_null`; do not stringify `"NULL"`.
- PostgreSQL placeholders start `$1`, MySQL/SQLite use `?`.
- Nox `Statement.bind_*` indexes start at zero on all drivers.
- Each `Statement` is single-use. Create a fresh statement per execution.
- Never share a mutable `Statement` across concurrent operations.
- Avoid hidden conversions and string interpolation of values.
- For repeated named values, choose deterministic parameter numbering.
- Reject unsupported value types rather than coercing unpredictably.
- Support explicit raw SQL with separate SQL string and typed bound values.

### Placeholder compiler invariants

For N parameters:
- SQLite/MySQL output contains N `?` placeholders in binding order.
- PostgreSQL output contains `$1` through `$N` in the exact same logical order.
- Bind indexes remain `0..N-1`.
- Parameters inside string literals/comments must not be counted as real placeholders when parsing user SQL; avoid rewriting arbitrary raw SQL by naive regex.
- Never substitute identifier names via value binders.

---

# 7. Dialect-specific SQL generation

Use a **dialect interface** for syntax that differs.

Required dialect decisions:
- identifier quoting and escaping;
- placeholder format;
- boolean encoding;
- auto-generated primary keys;
- `INSERT ... RETURNING` support;
- `LIMIT` / `OFFSET`;
- conflict/upsert syntax;
- transaction begin modes;
- schema/table qualification;
- SQL type names;
- reserved words and case sensitivity.

## 7.1 SQLite

- Parameter placeholders `?`.
- `:memory:` for fast unit tests.
- `last_insert_rowid` exists; its semantics are connection-scoped.
- SQLite type affinity and dynamic typing need explicit tests.
- `PRAGMA foreign_keys` state must be understood; never assume enabled.
- `BEGIN` / `COMMIT` behavior should be tested.
- Handle `SQLITE_BUSY` / locked database errors when distinguishable.
- Never copy a live SQLite file as a database backup.

## 7.2 PostgreSQL

- `$1`, `$2`, ... placeholders.
- Use `INSERT ... RETURNING` for generated keys.
- Respect transaction-aborted state after errors.
- Schema qualification, identifier case, `ILIKE`, JSON, `ON CONFLICT` are dialect-specific.
- Do not pretend SQLite SQL runs unmodified.
- Never log `conninfo` with credentials.
- `libpq` runtime availability must be checked.

## 7.3 MySQL / MariaDB

- `?` placeholders through documented Nox driver.
- The documented driver performs escaped substitution client-side; do not claim server-side prepare behavior.
- Auto-increment key behavior differs from PostgreSQL.
- Account for backtick quoting, boolean/TINYINT conventions, `LIMIT` syntax, upsert differences.
- Transaction support depends on storage engine and connection state.
- MySQL and MariaDB compatibility must be separately documented where they diverge.

## 7.4 Portability policy

- Provide a portable subset for common CRUD.
- Expose explicit dialect-specific extensions for nonportable features.
- Never silently rewrite unsupported SQL to something semantically different.
- Mark each feature with exact driver/dialect support and tests.
- Test SQL generation independently from DB execution.

---

# 8. Query builder and expression AST

Do not create a string-concatenation API that invites injection.

**MVP features:**
- `SELECT` with explicit columns.
- `WHERE` comparisons (`=`, `!=`, `<`, `<=`, `>`, `>=`).
- `AND` / `OR` grouping with explicit precedence.
- `IS NULL` / `IS NOT NULL`.
- `IN` with defined empty-list behavior.
- `ORDER BY` with validated column and direction.
- `LIMIT` / `OFFSET` with nonnegative bounds.
- `INSERT`, `UPDATE`, `DELETE`.
- Optional `COUNT`.

**Later features:**
- joins;
- grouping, `HAVING`, aggregates;
- pagination helpers;
- upserts;
- subqueries;
- CTEs;
- dialect-specific JSON operations.

### Safety invariants

- Column and table names come from validated metadata or an explicit trusted identifier type.
- Values always use bind parameters.
- Never accept raw user-supplied `ORDER BY`, `LIMIT` SQL fragments.
- Avoid SQL injection in `LIKE` patterns; escape wildcard characters if API promises literal matching.
- `IN []` must produce a safe false predicate or a documented error, never invalid SQL.
- `UPDATE` / `DELETE` without `WHERE` must require explicit confirmation/opt-in in safe high-level APIs.
- Preserve parentheses and operator precedence.
- Define NULL comparison semantics correctly (`IS NULL`, not `= NULL`).
- Bound nesting depth and query complexity where feasible.
- Do not claim SQL parser completeness.

### Raw SQL escape hatch

Allow advanced users to issue raw SQL using `DbConnection`/`Statement` or a thin Feather adapter.
- Require values to be passed separately and bound.
- Raw SQL is explicitly caller-controlled.
- Document identifier safety.
- Do not hide raw SQL from the user or force everything through the ORM.

---

# 9. CRUD behavior and result contracts

## Create

- Distinguish `insert one`, `insert many` (later), and upsert (later).
- Define which fields are included: defaults, generated IDs, nullable columns.
- Return affected row count and/or generated ID only when supported and unambiguous.
- No implicit `last_insert_rowid` on PostgreSQL.
- Do not assume a generated ID exists for every table.

## Read

- `find_by_pk`, `find_one`, `find_many`, `count` as ergonomic wrappers where implemented.
- Distinguish not-found from query error.
- Define ordering guarantees; absent `ORDER BY` means unspecified row order.
- No implicit `SELECT *` if stable column mapping requires an explicit projection.
- Avoid unbounded reads where callers request pagination; document full materialization when the driver returns `list[Row]`.

## Update

- Only update fields explicitly supplied unless a well-tested change tracker exists.
- Handle zero affected rows without inventing success.
- Prevent accidental updates of generated IDs or immutable columns.
- High-level full-table updates must require an explicit unsafe opt-in.

## Delete

- Define hard delete only for MVP.
- No automatic cascade behavior beyond database-defined constraints.
- Never hide deletion of multiple rows.
- High-level full-table deletes must require explicit unsafe opt-in.
- Soft delete is an optional application pattern, not implicit core behavior.

---

# 10. Transactions and units of work

**Transactions are critical but the Nox docs only establish generic `execute(sql)` and SQLite examples with `BEGIN`/`COMMIT`.** Verify each driver and dialect.

### MVP transaction goals

- Explicit `begin`, `commit`, `rollback` with clear ownership.
- Do not auto-commit on error.
- Roll back when operation fails and rollback is still possible.
- Do not swallow the original error if rollback also fails; report both when supported.
- No implicit nested transactions.
- Track transaction state: idle / active / committed / rolled back / failed.
- Prevent operations after connection close or invalid transaction state.
- Never close a caller-owned connection without an explicit ownership contract.

### Future transaction features

- Savepoints and nested transaction semantics.
- Isolation level configuration.
- Read-only transactions.
- Retry on serialization failures with idempotency checks.
- Unit-of-work batching and dirty tracking.

**Do not claim portable nested transactions, distributed transactions, or atomicity across separate connections.**

### Connection lifecycle

- Caller-owned connection vs Feather-owned connection must be explicit.
- Close Feather-owned connections on all supported paths.
- Never use `nox.os.exit` from transaction or driver code.
- Do not share connections across workers without documented safety.
- No global implicit connection/session.
- Connection pools are out of MVP and require independent concurrency design.

---

# 11. Relationships (later, intentionally limited)

Relationships can rapidly turn a lightweight ORM into a large framework. Do not start here.

If implemented:
- explicit foreign key metadata;
- one-to-one, one-to-many, many-to-one;
- joins or explicit eager loading;
- predictable query counts;
- no invisible network/database calls on property access;
- cycle-safe serialization;
- clear cascade behavior;
- explicit optionality and missing related-row behavior;
- documented composite-key limitations.

**Do not implement automatic lazy loading, polymorphic inheritance mapping, identity maps, or full relationship graph persistence without a separate approved design.**

---

# 12. Schema and migrations (future module)

Migrations are a **separate, optional capability**. Initial Feather must not auto-alter databases.

## Required migration design

- versioned migration identifiers;
- ordered up/down scripts or explicitly one-way migrations;
- applied-migration tracking table;
- checksum or equivalent change detection;
- schema/dialect compatibility checks;
- dry-run SQL preview;
- transaction strategy per dialect;
- explicit backup guidance;
- migration locking where feasible;
- idempotency and rerun behavior;
- error reporting and recovery instructions.

## Prohibited migration behavior

- No silent table drops, column drops, or destructive type conversions.
- No implicit migration on application startup by default.
- No fake rollback when DDL auto-commits on a particular engine.
- No claim of full cross-dialect migration portability.
- No schema diff engine that guesses user intent and deletes data.
- No production migration without explicit operator invocation.

For MVP, hand-written SQL migration files in the host application are acceptable.

---

# 13. Error model

Use real Nox exception syntax and available error-cause semantics.

Suggested **conceptual** hierarchy:
- `FeatherError`
- `ConnectionError`
- `QueryBuildError`
- `IdentifierError`
- `BindingError`
- `MappingError`
- `ConstraintError` (only if distinguishable)
- `TransactionError`
- `DialectError`
- `UnsupportedFeatureError`
- `MigrationError` (future)

Rules:
- Distinguish query construction failures from database execution failures.
- Preserve original driver error information where safe.
- Do not invent structured constraint error codes when driver only provides text.
- Never leak passwords, full DSNs, or secret query parameters in messages.
- Do not log all bound values by default.
- Include safe SQL context where helpful, with sensitive values redacted.
- Do not silently convert database failures into `None`/empty lists.
- Not-found is not automatically a database exception.

---

# 14. Noxen integration and validation

Feather must work **without** Noxen.

If Noxen is available later:
- provide a separate optional adapter for validating user input before persistence;
- distinguish Noxen validation errors from Feather mapping/database errors;
- do not reimplement all of Noxen;
- do not couple Feather's model metadata to Noxen's runtime availability;
- no automatic validation on reads unless explicitly configured;
- ensure database schema constraints remain authoritative.

The core Feather package should compile and run with no Noxen dependency.

---

# 15. Nyx and Aether integration

Feather is a standalone package. Integrations belong in optional adapters/examples.

For Nyx/Aether:
- explicit per-request connection/session lifecycle;
- never reuse an unsafe connection across independent worker tasks;
- transaction ownership at request boundaries;
- no automatic commit after a failed handler;
- predictable error-to-HTTP mapping in the adapter, not in core;
- connection strings from configuration, never hard-coded;
- do not add web framework imports to Feather core.

Aura and mobile targets:
- evaluate target-specific driver availability first;
- Nox docs state PostgreSQL/MySQL drivers are unavailable on mobile targets due to host shared-library loading;
- SQLite support depends on target/runtime capabilities and must be verified;
- do not promise mobile database support solely because core code compiles.

---

# 16. Security requirements

1. Bind all untrusted values via `Statement.bind_*`.
2. Validate and quote identifiers by dialect; identifiers cannot be bound as values.
3. Prevent accidental unfiltered UPDATE/DELETE in high-level APIs.
4. Never interpolate user input into schema, table, column, ORDER BY, or raw fragments.
5. Use least-privilege DB accounts in examples and production guidance.
6. Never store credentials in source, tests, logs, stack traces, or published artifacts.
7. Avoid exposing raw database error text to HTTP clients.
8. No automatic destructive migrations.
9. No SQL logging with secrets by default.
10. Limit unbounded queries in examples; encourage pagination.
11. Prevent path traversal when migration filenames or SQLite database paths are user-controlled.
12. Do not add native/FFI dependencies without review.
13. Audit all dialect escape functions and placeholder generation.
14. Review concurrency, transaction ownership, and rollback paths.
15. Document that parameterization protects **values**, not arbitrary SQL syntax.
16. Test malicious identifiers, quotes, Unicode, null bytes, comments, and SQL injection payloads.

---

# 17. Testing strategy

Use `nox.test` and `noxc test` as supported. Tests must run on the installed Nox compiler.

## 17.1 Unit tests (no external DB)

- Metadata validation: duplicate columns, invalid identifiers, no PK.
- SQL generation for all three dialects.
- Placeholder numbering and bind order.
- Empty IN lists, NULL comparisons, operator precedence.
- Query order, LIMIT/OFFSET boundaries.
- Insert/update/delete safety checks.
- Mapping: nullability, missing columns, invalid numeric conversion.
- Error sanitization and secret redaction.
- Connection ownership and transaction state-machine logic where mockable.
- Public API compile tests.

## 17.2 SQLite integration (required)

Use `nox.sqlite.open(":memory:")` where available.

- Create table; insert; read; update; delete.
- Typed values, SQL NULL, generated primary key.
- Multiple rows, empty results, explicit ORDER BY.
- Unique and foreign key constraints where enabled.
- Commit and rollback.
- Transaction error and recovery.
- Invalid SQL and malformed mapping.
- Parameterized SQL injection attempt.
- Close behavior and resource cleanup.
- Explicit raw SQL escape hatch.
- Driver/Feather parity for same queries.

## 17.3 PostgreSQL integration (required before claiming support)

Run against disposable local/test DB:
- `$1...$N` placeholder compilation and binding;
- `INSERT RETURNING`;
- transaction rollback after error;
- null/boolean/type behavior;
- identifier quoting;
- unique constraint behavior;
- schema qualification;
- query result mapping;
- connection error handling.

## 17.4 MySQL/MariaDB integration (required before claiming support)

Run against disposable test DB:
- `?` binding and escaping behavior;
- generated IDs;
- transactions on supported storage engine;
- identifier quoting;
- null/boolean/type behavior;
- charset/Unicode;
- rollback and connection errors.

## 17.5 CI rules

- No external database needed for fast core tests.
- PostgreSQL/MySQL CI jobs may use containers where available.
- A green SQLite suite **does not** establish PostgreSQL/MySQL support.
- Test claimed OS/backend combinations, including LLVM/QBE where relevant.
- Do not suppress, skip, or weaken tests to pass CI.
- Every bug fix gets a regression test.
- Use deterministic fixtures and isolate databases between tests.
- Never test destructive migrations against real production databases.

---

# 18. Performance and memory

- Benchmark Feather vs direct `nox.db` and `nox.orm` for CRUD.
- Measure SQL compile overhead, mapping overhead, batch sizes, and allocations.
- Record Nox version, backend, OS, database versions, iterations, schema, and query sizes.
- Document that `query()` returns `list[Row]`, implying full materialization in the documented API.
- Do not promise streaming cursors without driver support.
- Avoid repeated SQL parsing/compilation if a safe immutable query plan can be reused.
- **Do not cache executed `Statement` objects**: documented statements are single-use.
- Bound query AST depth and result size where practical.
- No hidden N+1 behavior.
- Connection pooling requires a separate verified concurrency design.
- Never sacrifice parameterization or transaction correctness for benchmark results.

---

# 19. Public API and stability

Each release must identify public vs internal surfaces.

- `docs/api.md`: documented public functions/types.
- `docs/compatibility.md`: Nox/compiler/driver/backend support.
- `docs/migration-guide.md`: breaking-change guidance.
- `CHANGELOG.md`: release history.

API stability labels:
- **Stable** — compatible within a major release.
- **Experimental** — subject to documented change.
- **Deprecated** — migration path and removal window.
- **Internal** — no compatibility promise.

Use SemVer:
- PATCH: compatible fixes.
- MINOR: compatible features.
- MAJOR: breaking stable API changes.

For 0.x releases, still document breaking changes. Add **compile-time API contract tests** for public examples. Keep the public API small.

### Suggested API shape (PSEUDOCODE ONLY)

```text
db = feather.connect_sqlite(":memory:")
users = feather.table("users", ...)
user = feather.insert(db, users, values)
result = feather.select(db, users).where(...).order_by(...).all()
feather.transaction(db, ...)
```

**This is not valid/verified Nox syntax.** Do not copy it into README until real code compiles. In particular, fluent chaining, callbacks, generic model return types, and connection constructors require feasibility checks.

---

# 20. Documentation and examples

`README.md` must explain:
- What Feather is, and what it is not.
- Difference between Feather and `nox.orm`.
- Verified Nox minimum version.
- Installation via real `nox.json` package syntax.
- SQLite quick start that compiles and runs.
- Parameterized CRUD.
- Model metadata and explicit mapping.
- Raw SQL usage.
- Transactions and connection cleanup.
- Dialect differences and supported features.
- Performance expectations and limitations.
- Security and license.

Additional docs:
- `architecture.md`: dependency direction.
- `capabilities.md`: evidence-backed Nox support matrix.
- `dialects.md`: SQLite/PostgreSQL/MySQL SQL differences.
- `transactions.md`: ownership and failure semantics.
- `security.md`: injection, identifiers, logging, credentials.
- `roadmap.md`: gated implementation milestones.

**Every executable Nox example must be compiled in CI.** Mark unverified designs clearly as pseudocode.

---

# 21. Package, release, and CI discipline

- Package name is `feather`, never `sable`.
- Use actual Nox package metadata (`nox.json`) and dependency locking conventions.
- Keep `nox.lock` in version control if generated.
- No mandatory third-party framework dependencies.
- Verify commands with installed toolchain:
  - `noxc version`
  - `noxc check <file.nox>`
  - `noxc run <file.nox>`
  - `noxc build <file.nox>`
  - `noxc test`
  - `noxc fmt <file.nox>` (modifies files in place)
- Do not invent compiler flags or CI commands.
- Do not publish, push tags, create releases, modify Nox stdlib, or edit other repositories without explicit user authorization.
- Do not update dependencies unnecessarily.
- Do not commit credentials, database files with real data, or test secrets.

---

# 22. Implementation roadmap

## Phase 0 — Feasibility and stdlib audit
Deliver:
- Verified `nox.db` / `nox.orm` / driver capability matrix.
- Proof of typed mapping and parameter binding.
- PostgreSQL placeholder strategy.
- Compiling package skeleton.
- Architecture and decisions docs.

**Gate:** all core design assumptions compile or have explicit alternatives.

## Phase 1 — Feather 0.1: SQLite-first core
Deliver:
- Typed metadata.
- Explicit row mapping.
- Safe CRUD.
- SQLite dialect.
- Basic expression builder.
- Raw SQL escape hatch.
- In-memory integration tests.

**Gate:** practical SQLite application works without framework dependencies.

## Phase 2 — Feather 0.2: multi-dialect correctness
Deliver:
- PostgreSQL dialect and integration tests.
- MySQL/MariaDB dialect and integration tests.
- Placeholder and identifier portability.
- Generated-ID differences.
- Feature matrix.

**Gate:** each claimed database works in real integration tests.

## Phase 3 — Feather 0.3: transactions and ergonomics
Deliver:
- Explicit transaction/session lifecycle.
- Reliable rollback and ownership rules.
- Improved errors and typed codecs.
- Additional query predicates.
- Public API stability review.

**Gate:** transaction failure tests pass for each claimed driver.

## Phase 4 — Feather 0.4: advanced queries
Deliver as justified:
- joins;
- aggregates and group-by;
- pagination;
- limited upsert;
- optional bulk operations;
- performance benchmarks.

**Gate:** no hidden N+1 queries and no unsafe SQL fragments.

## Phase 5 — Feather 0.5: optional integrations
Deliver:
- Nyx/Aether adapter examples.
- Optional Noxen validation adapter.
- Driver/target compatibility review.
- Stronger documentation and developer ergonomics.

**Gate:** core remains independent.

## Phase 6 — Feather 0.6+: migrations and relationships (optional)
Only after independent architecture review:
- explicit migration runner;
- schema version table;
- limited relations/eager loading;
- safe migration execution;
- cross-dialect behavior docs.

**Gate:** no destructive implicit schema changes or untested dialect assumptions.

## Phase 7 — Feather 1.0
- Stable minimal API.
- Comprehensive docs and examples.
- Security audit and performance comparison.
- CI for every claimed driver and platform.
- Migration guide and SemVer commitment.

**Gate:** all supported features are tested; unsupported features clearly documented; user approves release.

---

# 23. Task execution protocol for AI agents

For every task:

1. Restate scope and measurable acceptance criteria.
2. Inspect existing Feather code and Nox documentation/source.
3. Identify whether `nox.orm` already implements the requested behavior.
4. Confirm language features and driver capabilities with minimal compiling examples.
5. Describe the smallest viable design and public API impact.
6. Implement a vertical slice; do not rewrite unrelated code.
7. Add unit tests and appropriate database integration tests.
8. Run formatter, typecheck, tests, and examples.
9. Review SQL injection, identifier quoting, NULL behavior, transaction cleanup, and concurrency.
10. Update documentation, compatibility matrix, and changelog.
11. Report changed files, exact commands run, actual results, and unresolved limitations.

## When blocked

- Do not invent Nox syntax or pretend a driver capability exists.
- Provide a minimal failing reproduction.
- State the exact missing feature and which source proves the limitation.
- Offer a smaller supported alternative.
- Ask before native FFI, compiler changes, or major architectural scope expansion.
- Never mark untested PostgreSQL/MySQL support as complete.

---

# 24. Definition of Done

A feature is done only if:

- [ ] Valid, verified Nox code compiles.
- [ ] Public API is documented and minimally sufficient.
- [ ] SQL values are always bound.
- [ ] SQL identifiers are validated/quoted safely.
- [ ] Placeholder syntax is correct per dialect.
- [ ] NULL and typed conversion are correct.
- [ ] Transactions do not leak or silently commit.
- [ ] Connection ownership and worker safety are clear.
- [ ] Driver-specific behavior is tested where claimed.
- [ ] SQLite integration tests pass for portable core behavior.
- [ ] No framework coupling or unnecessary dependencies.
- [ ] No accidental `Sable` naming remains.
- [ ] Examples compile; docs and changelog are current.
- [ ] No test failures hidden or skipped.
- [ ] Final report honestly states what was verified.

---

# 25. First autonomous agent assignment

**Implement Feather's real foundation; do not attempt a giant ORM in one pass.**

1. Audit Nox `nox.db`, `nox.orm`, SQLite/PostgreSQL/MySQL drivers and Nox type/reflection capabilities.
2. Record exact compiler version and documented API signatures.
3. Explain the differences between Feather's proposed value and the existing `nox.orm`.
4. Create a valid independent `feather` package and documentation skeleton.
5. Build a working SQLite `:memory:` CRUD example using prepared parameters.
6. Add explicit model metadata and row mapping for one simple typed Nox model.
7. Add a minimal safe query builder for equality, AND, ORDER BY, LIMIT.
8. Implement SQLite SQL generation and test PostgreSQL/MySQL placeholder generation independently.
9. Add transaction proof-of-concept and tests only after confirming driver semantics.
10. Publish an evidence-based backlog for other drivers, relationships, migrations, pooling, and advanced types.
11. Do not alter the Nox compiler/stdlib or publish a package without asking.

**Quality standard:** a small, truly working Nox-native mapper and SQL builder is more valuable than an enormous API that merely resembles Python ORMs.
