# Security Policy

## Supported Versions

We actively maintain and provide security updates for the current major/minor line of Feather.

| Version | Supported          | Status |
| ------- | ------------------ | ------ |
| 0.2.x   | :white_check_mark: | Active support (current stable release line) |
| 0.1.x   | :x:                | End of life (please upgrade to 0.2.8+) |

---

## Core Security Invariants in Feather

Feather is built with strict security invariants designed to eliminate SQL injection, race conditions, credential leaks, and data corruption by construction:

1. **Always-Bound Parameter Values:**
   Untrusted user values are never concatenated into SQL query text. All values pass through `Statement.bind_*` parameters with 0-based indices and deterministic type mapping.

2. **Identifier Validation and Quoting:**
   SQL identifiers (table names, column names) cannot be bound as parameter values in standard SQL engines. Feather validates identifiers against strict character patterns (`[a-zA-Z_][a-zA-Z0-9_]*`) and automatically quotes identifiers per dialect (`"` for SQLite/PostgreSQL, `` ` `` for MySQL).

3. **Protection Against Accidental Bulk Mutation:**
   High-level `update` and `delete` operations require explicit `WhereClause` predicates or explicit target IDs to prevent unintended table-wide modification.

4. **Credential and Secret Redaction:**
   Error reporting sanitizes sensitive connection details, credentials, and raw bound values to prevent password or token leakage in logs and exception traces.

5. **Single-Use Statement Hygiene:**
   Prepared statements in Nox are single-use (`step` followed by `finalize`). Feather creates fresh statements per execution and prevents reuse of closed handles.

6. **Poisoned Session Quarantine:**
   Sessions that experience socket drops or failed commits/rollbacks are actively marked `is_poisoned = True`, preventing further query execution and isolating corrupted connections from connection pools.

7. **Cryptographic Migration Locking & Lease Fencing:**
   Schema migrations use atomic database updates with cryptographically seeded UUID tokens (`generate_lock_token()`), active statement-level lease fencing, and engine-level advisory locking (`pg_try_advisory_lock`, `GET_LOCK`), preventing concurrent schema mutation or split-brain execution across multiple workers.

---

## Reporting a Vulnerability

If you discover a potential security vulnerability in Feather, please follow our responsible disclosure process:

- **Do NOT open a public GitHub issue.**
- Email the maintainer directly at **`mburakmemis@gmail.com`** with the subject `[SECURITY] Feather Vulnerability Report`.
- Include:
  - Description of the vulnerability and its potential impact.
  - Minimal reproducible example or proof-of-concept in Nox.
  - Affected Feather versions and target database engines.
- We will acknowledge receipt within **24 hours**, provide a coordinated remediation timeline, and credit your report in the security release advisory once patched.
