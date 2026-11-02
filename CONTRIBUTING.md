# Contributing to Feather 🪶

Thank you for your interest in contributing to **Feather**! We welcome contributions that help improve performance, expand dialect fidelity, enhance documentation, or fix bugs while keeping the library compact, lightweight, and type-safe.

---

## Code of Conduct

All contributors and maintainers are expected to follow our [Code of Conduct](CODE_OF_CONDUCT.md). Please report unacceptable behavior to `mburakmemis@gmail.com`.

---

## Guiding Principles

Feather follows strict architectural principles defined in [`AGENTS.md`](AGENTS.md) and [`docs/decisions.md`](docs/decisions.md):

1. **SQL-First, Not SQL-Hostile:** Raw SQL is a first-class escape hatch, not a failure.
2. **Explicit Static Typing:** Zero runtime metaclasses, monkey-patching, or fictional reflection. Every function, parameter, and field must have explicit static types.
3. **Security by Construction:** Values are always bound as prepared statement parameters. Identifiers are strictly validated and dialect-quoted.
4. **Zero External Dependencies:** Feather builds directly on the standard Nox library (`nox.db`, `nox.sqlite`, `nox.postgres`, `nox.mysql`, `nox.strings`, `nox.crypto`, `nox.random`, `nox.time`, `nox.test`).
5. **Dialect Honesty:** Never assume all engines behave identically (e.g. MySQL non-transactional DDL, SQLite parameter limits, PostgreSQL `$1` placeholders).

---

## Local Development Setup

### 1. Prerequisites

Ensure the Nox compiler (`noxc`) is installed:

```bash
noxc version
```

If not installed, install it via the official installer:
```bash
# macOS / Linux
curl -fsSL https://raw.githubusercontent.com/mburakmmm/nox-lang/main/install.sh | bash

# Windows (PowerShell)
irm https://raw.githubusercontent.com/mburakmmm/nox-lang/main/install.ps1 | iex
```

### 2. Clone the Repository

```bash
git clone https://github.com/mburakmmm/feather.git
cd feather
```

### 3. Verify Local Typecheck & Tests

```bash
# Static typecheck
noxc check feather.nox
noxc check main.nox

# Run full test suite (30 test suites across unit & integration tests)
noxc test

# Run demo verification
noxc run main.nox
```

All core tests run against in-memory SQLite fixtures (`:memory:` or `file:memdb_shared?mode=memory&cache=shared`) without requiring external network database services.

---

## Branching & Commit Guidelines

- **Branch naming:** Use descriptive names like `fix/advisory-lock-parsing`, `feat/composite-indexes`, `docs/migration-guide`.
- **Commit messages:** Follow the [Conventional Commits](https://www.conventionalcommits.org/) specification:
  - `feat: ...` for new capabilities
  - `fix: ...` for bug fixes
  - `docs: ...` for documentation updates
  - `test: ...` for test suite additions
  - `perf: ...` for performance improvements

---

## Submitting a Pull Request

1. Fork the repository and create your branch from `main`.
2. Implement your changes following Nox idiomatic syntax.
3. Add unit or integration tests under `tests/unit/` or `tests/integration/`.
4. Ensure `noxc test` passes 100% locally.
5. Update `CHANGELOG.md` under an `[Unreleased]` or targeted version section.
6. Submit your PR against `main` using the provided [Pull Request Template](.github/PULL_REQUEST_TEMPLATE.md).
7. Ensure all matrix checks pass in the GitHub Actions CI pipeline.
