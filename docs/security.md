# Security Invariants and Threat Model (docs/security.md)

Feather is designed from the ground up to prevent the most common SQL security vulnerabilities.

## 1. Value Parameterization
Untrusted user data is **never** concatenated directly into SQL text. Every value is represented via the typed `Value` union and bound to a single-use `Statement` using zero-based indices.

## 2. Identifier Validation and Quoting
SQL identifiers (table names, column names) cannot be bound through database driver value binders. Feather mitigates identifier injection by:
- Enforcing strict ASCII identifier regex validation (`[a-zA-Z_][a-zA-Z0-9_]*`) on all tables and columns.
- Rejecting malicious characters (quotes, semicolons, dashes, spaces, null bytes, SQL keywords).
- Quoting identifiers with dialect-specific delimiters (`"` for SQLite/PostgreSQL, `` ` `` for MySQL).

## 3. Accidental Bulk Update / Delete Prevention
Unfiltered `UPDATE` and `DELETE` queries executed through high-level APIs raise `QueryBuildError` unless an explicit non-empty `WhereClause` is supplied.

## 4. Credential Sanitization
All connection strings and error messages are passed through `sanitize_error_message()`, which strips passwords, tokens, and sensitive query fragments before exposing them in logs or exception traces.
