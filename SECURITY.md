# Security Policy

## Supported Versions

| Version | Supported          |
| ------- | ------------------ |
| 0.1.x   | :white_check_mark: |

## Security Invariants in Feather

Feather is built with strict security invariants to eliminate SQL injection and credential leaks by design:

1. **Always-Bound Parameter Values:**
   Untrusted user values are never concatenated into SQL query text. All values pass through `Statement.bind_*` parameters with 0-based indices.

2. **Identifier Validation and Quoting:**
   SQL identifiers (table names, column names) cannot be bound as parameter values in standard SQL engines. Feather validates identifiers against strict character patterns (`[a-zA-Z_][a-zA-Z0-9_]*`) and automatically quotes identifiers per dialect (`"` for SQLite/PostgreSQL, `` ` `` for MySQL).

3. **Protection Against Accidental Bulk Mutation:**
   High-level `update` and `delete` operations require explicit `WhereClause` predicates or explicit target IDs to prevent unintended table-wide modification.

4. **Credential and Secret Redaction:**
   Error reporting sanitizes sensitive connection details, credentials, and raw bound values to prevent password or token leakage in logs and exception traces.

5. **Single-Use Statement Hygiene:**
   Prepared statements in Nox are single-use (`step` followed by `finalize`). Feather creates fresh statements per execution and prevents reuse of closed handles.

## Reporting a Vulnerability

If you discover a security vulnerability in Feather:
- Do not open a public issue.
- Please email `mburakmemis@gmail.com` with reproduction steps and details.
- We will respond within 48 hours and coordinate a security fix.
