# Compatibility Matrix (docs/compatibility.md)

This document tracks verified compatibility across compiler versions, database engines, and runtime operating systems.

## Compiler Support

| Compiler Version | Tested Date | Status |
| ---------------- | ----------- | ------ |
| `noxc 2.2.38`   | 2026-11-19  | Verified Pass (:white_check_mark:) |
| `noxc 2.1.186`   | 2026-11-15  | Verified Pass (v0.3.0 line) |
| `noxc 2.0.27`    | 2026-10-27  | Verified Pass (v0.2.x line) |

## Database Engine Drivers

| Database | Driver | Status | Notes |
| -------- | ------ | ------ | ----- |
| SQLite 3 | `nox.sqlite` | Verified Pass (:white_check_mark:) | Tested with `:memory:`, `safe_sqlite_path`, and multi-statement `execute_script()` (`nox.sqlite` 2.1.195+) across all unit & integration tests. |
| PostgreSQL | `nox.postgres` | Dialect + Error/Timeout Ready | Verified SQL compilation (`$1..$N`, `RETURNING`, `SERIAL`), `PostgresError` field mapping (`sqlstate`, `table`, `column`, `constraint`), and query/statement/lock timeouts & `cancel()` (including pure-Zig mobile `CancelRequest` on iOS/Android since `2.2.2` and WASM host read timeouts since `2.2.4`). External PostgreSQL server required for live DB execution. |
| MySQL / MariaDB | `nox.mysql` | Dialect Ready | Verified SQL compilation (backtick quotes, `?` placeholders, `INT AUTO_INCREMENT`). External MySQL/MariaDB server required for live DB execution. |

## Platform & Target Policy

- **macOS (Darwin ARM64 / x86_64):** Fully tested and passing on `noxc 2.1.186`.
- **Linux (x86_64 / aarch64):** Fully supported via standard POSIX runtime and dynamic `libsqlite3` / `libpq` / `libmysqlclient` loading.
- **Windows:** Supported via Windows C runtime and dynamic DLL resolution.
- **WebAssembly / Mobile (`wasm32-*`, `ios-*`, `android-*`):** Feather's core modules (value codecs, query AST, dialect SQL compilers, Pure-Nox JSON codecs, `nox.reflect` schema builder) are pure Nox with zero C-backed `nox.xxhash`, `nox.yyjson`, or `nox.pcre2` dependencies. Note that native database drivers (`nox.postgres`, `nox.mysql`, host `libsqlite3`) require a hosted runtime or virtual filesystem adapter depending on the target profile.
