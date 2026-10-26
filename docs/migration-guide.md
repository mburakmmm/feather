# Migration Guide: Upgrading and Transitioning to Feather (docs/migration-guide.md)

This guide helps developers migrate existing raw SQL or `nox.orm` codebases to Feather.

## Migrating from `nox.orm` to Feather

### 1. Table Definitions
In `nox.orm`:
```nox
col: Column = Column("id", 0, True) # 0 was INTEGER
tbl: Table = Table("users", [col])
```

In `feather`:
```nox
import feather
from feather import col_int, col_str, table, Table

tbl: Table = table("users", [
    col_int("id", True, False, True),
    col_str("name")
])
```
*Benefits:* Explicit keyword parameters, identifier validation, duplicate column prevention, and dialect awareness.

### 2. Inserting Records
In `nox.orm`:
```nox
orm.insert(conn, tbl, {"name": orm.val_str("Alice")})
```

In `feather`:
```nox
s: Session = feather.connect_sqlite("app.db")
new_id: int = s.insert(tbl, {"name": val_str("Alice")})
```
*Benefits:* Returns the auto-generated primary key ID directly on supported engines.

### 3. Querying Records
In `nox.orm`:
```nox
rows = orm.select(conn, tbl, "age > ?", [orm.val_int(18)])
# Only returned raw Row, requiring manual indexing and untyped field access
```

In `feather`:
```nox
q: SelectQuery = select_all("users").where(gt("age", val_int(18)))
users: list[User] = s.execute_query[User](user_model, q)
```
*Benefits:* Type-safe generic model mapping (`list[User]`), dialect portability ($1..$N on PostgreSQL), and protection against SQL injection.
