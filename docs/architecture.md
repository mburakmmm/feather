# Feather Architecture (docs/architecture.md)

Feather is built as a layered, modular SQL data mapper and query toolkit for the Nox programming language.

## Layer Diagram

```text
               +-----------------------------------------+
               |        Application / User Code          |
               +-----------------------------------------+
                                    |
               +-----------------------------------------+
               |        Feather Public Facade            |
               |             (feather.nox)               |
               +-----------------------------------------+
                 /          |           \              \
                /           |            \              \
    +--------------+ +--------------+ +---------------+ +---------------+
    |  Model &     | |    Query     | |   Session &   | |  Migrations & |
    |  Metadata    | |   Builder    | |  Transaction  | |  Relations    |
    | (mapper.nox) | | (query.nox)  | | (session.nox) | |(migrations.nox|
    | (metadata.nox|                | |(transact.nox) | | relations.nox)|
    +--------------+ +--------------+ +---------------+ +---------------+
            \               |                /
             \              |               /
         +--------------------------------------+
         |            Dialect Layer             |
         |        (SQLite / Postgres / MySQL)   |
         |            (dialects.nox)            |
         +--------------------------------------+
                            |
         +--------------------------------------+
         |       Typed Values & Parameters      |
         |             (values.nox)             |
         +--------------------------------------+
                            |
         +--------------------------------------+
         |        Nox Standard Database         |
         |   (nox.db / sqlite / postgres / mysql|
         +--------------------------------------+
```

## Module Responsibilities

1. **`errors.nox`:**
   Hierarchical error classes (`FeatherError`, `ConnectionError`, `QueryBuildError`, `ConstraintError`, `TransactionError`, etc.) and sensitive credential/value sanitizer.

2. **`values.nox`:**
   Typed SQL value union (`Value`), literal constructors (`val_int`, `val_str`, `val_float`, `val_bool`, `val_null`), and statement binding dispatcher.

3. **`metadata.nox`:**
   Table and column schema declarations (`Table`, `Column`), data types, primary key flags, nullability flags, and identifier validation.

4. **`dialects.nox`:**
   SQL generation specifics per engine: identifier quotation, placeholder formatting (`?` vs `$1..$N`), DDL type mappings, and limit/offset formatting.

5. **`query.nox`:**
   Type-safe AST and fluent builder for `SELECT`, `WHERE` predicates (`eq`, `neq`, `gt`, `gte`, `lt`, `lte`, `like`, `is_null`, `is_not_null`, `is_in`), joins (`inner_join`, `left_join`), ordering, pagination, and SQL compilation into parameterized `CompiledQuery`.

6. **`mapper.nox`:**
   Generic `Model[T]` binding metadata with typed row decoder `(Row) -> T` and encoder `(T) -> dict[str, Value]`.

7. **`session.nox`:**
   Unit of work coordinating connection execution, CRUD operations, parameterized raw queries, and DDL execution.

8. **`transaction.nox`:**
   Transaction state tracking (`IDLE`, `ACTIVE`, `COMMITTED`, `ROLLED_BACK`, `FAILED`) and safe transaction execution scopes with rollback on error.

9. **`pool.nox`:**
   Connection pool managing active vs idle connections with bounded capacity and cleanup.

10. **`migrations.nox`:**
    Schema migration manager tracking applied migration versions, verifying status, and running up/down scripts inside transactions.

11. **`relations.nox`:**
    Relationship metadata (`HasOne`, `HasMany`, `BelongsTo`) and transparent eager-loading query helpers.
