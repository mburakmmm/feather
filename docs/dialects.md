# Database Dialects (docs/dialects.md)

Feather supports three major SQL database engines through dedicated dialect definitions:

## Comparison Matrix

| Feature | SQLite | PostgreSQL | MySQL / MariaDB |
| ------- | ------ | ---------- | --------------- |
| Placeholder | `?` | `$1`, `$2`, ... | `?` |
| Identifier Quoting | `"identifier"` | `"identifier"` | `` `identifier` `` |
| Auto-Increment DDL | `INTEGER PRIMARY KEY AUTOINCREMENT` | `SERIAL PRIMARY KEY` | `INT AUTO_INCREMENT PRIMARY KEY` |
| Generated Primary Key | `last_insert_rowid()` | `INSERT ... RETURNING <pk>` | `last_insert_rowid()` (`mysql_insert_id`) |
| Boolean Type DDL | `INTEGER` | `BOOLEAN` | `TINYINT(1)` |
| Real / Float Type DDL | `REAL` | `DOUBLE PRECISION` | `DOUBLE` |
| Transaction Start | `BEGIN` | `BEGIN` | `START TRANSACTION` |
| Transactional DDL | Yes (supports DDL rollback) | Yes (supports DDL rollback) | No (DDL causes implicit commit) |
| Migration Lock Row Init | `INSERT OR IGNORE` | `ON CONFLICT (id) DO NOTHING` | `ON DUPLICATE KEY UPDATE` |
| Migration Tracking Table | Includes `status` column | Includes `status` column | Includes `status` column (`RUNNING`/`APPLIED`/`FAILED`) |

## Dialect Methods for Migrations & Concurrency

- `supports_transactional_ddl() -> bool`: Returns `True` for SQLite and PostgreSQL where DDL statements can be executed inside transactions and rolled back. Returns `False` for MySQL where DDL causes implicit transaction commits.
- `migration_table_sql() -> str`: Returns dialect DDL for `_feather_migrations` with `status TEXT NOT NULL DEFAULT 'APPLIED'`.
- `migration_lock_table_sql() -> str`: Returns dialect DDL for `_feather_migration_lock` with `timeout_at` for stale lock recovery.
- `init_migration_lock_row_sql() -> str`: Pre-initializes row `id = 1` atomically across dialects so that concurrent workers can compete via a single conditional `UPDATE`.

## Instantiating Dialects

```nox
import feather
from feather import Dialect, sqlite_dialect, postgres_dialect, mysql_dialect

d_sqlite: Dialect = sqlite_dialect()
d_pg: Dialect = postgres_dialect()
d_my: Dialect = mysql_dialect()
```
