# Schema Migrations (docs/migrations.md)

Feather provides a versioned, transactional migration system to evolve database schemas safely.

## Migration Structure

Each migration consists of:
- `version: int`: A unique positive integer defining sequence order (e.g., `1`, `2`, `3`).
- `name: str`: Descriptive title for the migration.
- `up_sqls: list[str]`: List of DDL statements to apply forward.
- `down_sqls: list[str]`: List of DDL statements to roll back.

## Defining and Running Migrations

```nox
import feather
from feather import Session, Migration, MigrationManager, migration, open_sqlite_memory

s: Session = open_sqlite_memory()

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

# Apply all pending migrations forward
applied_count: int = mgr.migrate_up()
print("Applied migrations: " + str(applied_count))

# Print migration status report
status_lines: list[str] = mgr.status()
i: int = 0
while i < len(status_lines):
    print(status_lines[i])
    i = i + 1

# Roll back to version 0 (complete rollback)
mgr.migrate_down(0)
```

## Migration Tracking Table

Feather stores applied migration metadata in `_feather_migrations`:
- `id`: Auto-incrementing internal sequence key.
- `version`: Unique migration integer version.
- `name`: Migration name.
- `applied_at`: Timestamp recording when the migration executed.

Each migration step is executed within an isolated database transaction. If any statement fails, the transaction is rolled back completely.
