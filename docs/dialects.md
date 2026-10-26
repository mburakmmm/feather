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

## Instantiating Dialects

```nox
import feather
from feather import Dialect, sqlite_dialect, postgres_dialect, mysql_dialect

d_sqlite: Dialect = sqlite_dialect()
d_pg: Dialect = postgres_dialect()
d_my: Dialect = mysql_dialect()
```
