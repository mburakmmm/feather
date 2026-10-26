# Compatibility Matrix (docs/compatibility.md)

This document tracks verified compatibility across compiler versions, database engines, and runtime operating systems.

## Compiler Support

| Compiler Version | Tested Date | Status |
| ---------------- | ----------- | ------ |
| `noxc 2.0.27`    | 2026-10-27  | Verified Pass (:white_check_mark:) |

## Database Engine Drivers

| Database | Driver | Status | Notes |
| -------- | ------ | ------ | ----- |
| SQLite 3 | `nox.sqlite` | Verified Pass (:white_check_mark:) | Tested with in-memory (`:memory:`) fixtures across all CRUD, query, and transaction tests. |
| PostgreSQL | `nox.postgres` | Dialect Ready (:white_check_mark:) | Verified SQL compilation ($1..$N placeholders, RETURNING, SERIAL). Full integration requires external container. |
| MySQL / MariaDB | `nox.mysql` | Dialect Ready (:white_check_mark:) | Verified SQL compilation (backtick quotes, ? placeholders, INT AUTO_INCREMENT). |

## Operating System Compatibility

- **macOS (Darwin ARM64 / x86_64):** Fully tested and passing.
- **Linux (x86_64 / aarch64):** Fully supported via standard POSIX runtime and system dynamic libraries.
- **Windows:** Supported via Windows C runtime and dynamic DLL resolution.
