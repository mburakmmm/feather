# Feather Implementation Roadmap (docs/roadmap.md)

## Phase 0: Feasibility and Stdlib Audit (:white_check_mark:)
- [x] Audit `nox.db`, `nox.sqlite`, `nox.postgres`, `nox.mysql`.
- [x] Verify generic classes `class Box[T]` and method generics in Nox.
- [x] Verify union types `str | None` and local variable type narrowing.
- [x] Document language limitations and architectural decisions.

## Phase 1: Core Data Mapper & SQLite Foundation (:white_check_mark:)
- [x] Type-safe `Value` union and zero-based parameter binding.
- [x] Declarative table and column schema metadata (`Table`, `Column`).
- [x] Generic `Model[T]` with pure `(Row) -> T` decoder and `(T) -> dict[str, Value]` encoder.
- [x] Full CRUD operations (`insert`, `find_by_id`, `find_all`, `find_where`, `update_by_id`, `delete_by_id`, `count`).
- [x] High-level bulk update/delete protection against unfiltered table mutations.
- [x] SQLite in-memory integration test suite.

## Phase 2: Multi-Dialect Query Engine (:white_check_mark:)
- [x] SQLite dialect (`?`, double quote, `INTEGER PRIMARY KEY AUTOINCREMENT`).
- [x] PostgreSQL dialect (`$1..$N`, double quote, `SERIAL PRIMARY KEY`, `INSERT ... RETURNING`).
- [x] MySQL dialect (`?`, backtick quotes, `INT AUTO_INCREMENT PRIMARY KEY`).
- [x] AST expression builder (`eq`, `neq`, `gt`, `gte`, `lt`, `lte`, `like`, `is_null`, `is_not_null`, `is_in`).
- [x] Safe empty IN list compilation (`1 = 0`).
- [x] Inner and left joins.

## Phase 3: Transactions & Unit of Work (:white_check_mark:)
- [x] State-tracked `Transaction` (`IDLE`, `ACTIVE`, `COMMITTED`, `ROLLED_BACK`, `FAILED`).
- [x] Transactional rollback on error without swallowing original exceptions.
- [x] Session ownership semantics and connection lifecycle management.

## Phase 4: Connection Pooling & Migrations (:white_check_mark:)
- [x] Bounded `ConnectionPool` with `acquire()` and `release_connection()`.
- [x] Schema migration manager (`MigrationManager`) with version tracking.
- [x] Idempotent forward (`up`) and backward (`down`) migrations.
- [x] Status reporting for migrations.

## Phase 5: Relationship Loaders (:white_check_mark:)
- [x] Declarative `has_one`, `has_many`, `belongs_to` metadata.
- [x] `RelationLoader` for explicit, transparent eager loading without N+1 queries.

## Phase 6: Ecosystem Hardening & 1.0 Release
- [ ] Containerized multi-engine CI matrix (PostgreSQL / MySQL live test suite).
- [ ] Automated CLI migration runner command.
- [ ] Benchmark suite vs raw `nox.db`.
