# Connection Pooling (docs/pooling.md)

Feather includes a lightweight, bounded connection pool for reusing open database handles efficiently.

## Overview

Opening a new database connection for every operation or web request can introduce significant latency and file descriptor churn. `ConnectionPool` maintains a queue of idle, pre-opened connections and enforces a strict upper capacity limit.

## Usage

```nox
import feather
from feather import ConnectionPool, Session, create_sqlite_pool

# Create pool with up to 5 connections
pool: ConnectionPool = create_sqlite_pool("production.db", 5)

# Acquire session
s: Session = pool.acquire()

# Run database operations
s.execute_raw("INSERT INTO metrics VALUES (1)", [])

# Return connection to pool
pool.release_connection(s)

# Teardown pool when shutting down
pool.close_all()
```

## Pool Invariants

1. **Upper Bound Safety:** Acquiring beyond `max_connections` raises `PoolError` rather than silently leaking memory.
2. **Session Decoupling:** Closing a borrowed `Session` does not destroy the pooled connection handle unless the pool itself is closed.
3. **Graceful Teardown:** `close_all()` releases all cached idle connection handles.
