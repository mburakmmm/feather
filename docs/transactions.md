# Transactions and Unit of Work (docs/transactions.md)

Feather provides explicit transaction lifecycle tracking with automatic failure safety.

## Transaction States

A transaction transitions through strict states:
```text
           +----------------+
           |  TX_STATE_IDLE |
           +----------------+
                   |
                begin()
                   v
          +------------------+
          | TX_STATE_ACTIVE  |
          +------------------+
             /            \
         commit()       rollback()
           /                \
          v                  v
+--------------------+   +-----------------------+
| TX_STATE_COMMITTED |   | TX_STATE_ROLLED_BACK  |
+--------------------+   +-----------------------+
```

If an error occurs while active, the transaction enters `TX_STATE_FAILED` and must be rolled back.

## Idiomatic Transaction Usage

```nox
import feather
from feather import Session, Transaction, begin_transaction, val_int

s: Session = feather.connect_sqlite("app.db")
tx: Transaction = begin_transaction(s)
try:
    s.update_by_id(accounts_tbl, val_int(1), {"balance": val_int(800)})
    s.update_by_id(accounts_tbl, val_int(2), {"balance": val_int(700)})
    tx.commit()
    print("Transaction committed successfully.")
except Exception as e:
    tx.rollback()
    print("Transaction rolled back due to error: " + e.message)
```

## Invariants

1. **No Silent Commit:** Errors never commit automatically.
2. **Explicit Rollback:** Failed transactions can be rolled back without swallowing the original exception.
3. **No Implicit Nesting:** Savepoints and nested transactions require explicit configuration; transactions are connection-scoped.
