# Performance and Resource Guidelines (docs/performance.md)

Feather is engineered for low latency, predictable allocations, and safe memory management in Nox.

## 1. Single-Use Statement Hygiene
Nox `Statement` handles are single-use (`step` followed by immediate `finalize`). Feather creates fresh statements on demand and frees them immediately, avoiding dangling handles and file descriptor leaks.

## 2. Full Materialization Contract
Because the underlying Nox drivers return complete `list[Row]` structures from queries, Feather materializes rows synchronously into typed models. For large datasets, developers should always use `.limit()` and `.offset()` for pagination rather than unbounded reads.

## 3. Zero-Allocation AST Compilations
Query compilation uses efficient string concatenations and vector appends. Parameter arrays are pre-sized where practical to minimize reallocations.

## 4. Connection Pooling Efficiency
Using `ConnectionPool` avoids the overhead of repeated OS `open()` and `close()` syscalls, significantly improving throughput for web applications and CLI microservices.
