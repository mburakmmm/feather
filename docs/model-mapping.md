# Model Mapping in Feather (docs/model-mapping.md)

Feather uses an explicit, type-safe mapping pattern tailored to the Nox programming language.

## Design Philosophy

Unlike dynamic ORMs that mutate classes at runtime using metaclasses or macros, Feather separates:
1. **The Domain Entity:** A pure typed Nox class representing application data.
2. **The Table Schema (`Table`):** Declarative SQL metadata including column types, constraints, and keys.
3. **The Decoder / Encoder Functions:** Pure, testable functions that convert `Row <-> Entity`.
4. **The `Model[T]`:** A lightweight generic binding coordinating table metadata with row conversion.

## Defining an Entity and Model

```nox
import feather
from feather import Table, Column, Model, Value, val_int, val_str, val_bool, col_int, col_str, col_bool, table, row_get_int, row_get_str, row_get_bool
import nox.db
from nox.db import Row

class User:
    id: int
    name: str
    email: str
    is_active: bool

    def __init__(self: User, id: int, name: str, email: str, is_active: bool) -> None:
        self.id = id
        self.name = name
        self.email = email
        self.is_active = is_active

def decode_user(r: Row) -> User:
    return User(
        row_get_int(r, 0),
        row_get_str(r, 1),
        row_get_str(r, 2),
        row_get_bool(r, 3)
    )

def encode_user(u: User) -> dict[str, Value]:
    d: dict[str, Value] = {}
    d["id"] = val_int(u.id)
    d["name"] = val_str(u.name)
    d["email"] = val_str(u.email)
    d["is_active"] = val_bool(u.is_active)
    return d

user_tbl: Table = table("users", [
    col_int("id", True, False, True),
    col_str("name"),
    col_str("email"),
    col_bool("is_active")
])

user_model: Model[User] = Model[User](user_tbl, decode_user, encode_user)
```

## Supported Types and Row Getters

| SQL Type | Column Constructor | Row Getter | Optional Getter |
| -------- | ------------------ | ---------- | --------------- |
| `INTEGER` | `col_int(name)` | `row_get_int(row, idx)` | `row_get_opt_int(row, idx)` |
| `TEXT` / `VARCHAR` | `col_str(name)` | `row_get_str(row, idx)` | `row_get_opt_str(row, idx)` |
| `REAL` / `FLOAT` | `col_float(name)` | `row_get_float(row, idx)` | `row_get_opt_float(row, idx)` |
| `BOOLEAN` | `col_bool(name)` | `row_get_bool(row, idx)` | `row_get_opt_bool(row, idx)` |

Nullable fields use Nox union types (`str | None`, `int | None`). When a database column contains SQL `NULL`, the optional getters return `None`.
