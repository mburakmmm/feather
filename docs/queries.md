# Query Construction and Expression AST (docs/queries.md)

Feather provides an AST-based query builder that produces parameterized SQL.

## Basic Query Building

```nox
import feather
from feather import select, select_all, SelectQuery, CompiledQuery, eq, gt, val_int, val_str, sqlite_dialect

# SELECT * FROM "users" WHERE "age" > ? AND "status" = ?
q: SelectQuery = select_all("users").where(gt("age", val_int(18))).and_where(eq("status", val_str("active")))
compiled: CompiledQuery = q.compile(sqlite_dialect())
```

## Supported Predicates

- `eq(col, val)`: Equality (`col = ?`)
- `neq(col, val)`: Inequality (`col != ?`)
- `gt(col, val)`: Greater than (`col > ?`)
- `gte(col, val)`: Greater than or equal (`col >= ?`)
- `lt(col, val)`: Less than (`col < ?`)
- `lte(col, val)`: Less than or equal (`col <= ?`)
- `like(col, pattern)`: SQL LIKE (`col LIKE ?`)
- `is_null(col)`: SQL NULL check (`col IS NULL`)
- `is_not_null(col)`: SQL NOT NULL check (`col IS NOT NULL`)
- `is_in(col, vals)`: SQL IN list (`col IN (?, ?)`). An empty list safely compiles to `1 = 0`.

## Joins, Sorting, and Pagination

```nox
q: SelectQuery = select_all("orders")
    .inner_join("users", "user_id", "id")
    .where(eq("orders.status", val_str("shipped")))
    .order_by("orders.created_at", ORDER_DESC)
    .limit(10)
    .offset(20)
```

## Compilation Guarantees

1. **Identifier Quoting:** Table and column names are automatically quoted according to the active dialect (`"` for SQLite/PostgreSQL, `` ` `` for MySQL).
2. **Deterministic Placeholder Order:** Parameters are bound in the exact logical order in which they appear in the query AST.
3. **PostgreSQL Compatibility:** When compiling for PostgreSQL, placeholders automatically number from `$1` to `$N`.
