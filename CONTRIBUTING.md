# Contributing to Feather

Thank you for contributing to Feather, the lightweight SQL data mapper and query toolkit for Nox.

## Development Setup

1. Ensure the Nox compiler (`noxc`) is installed and available on your PATH:
   ```bash
   noxc version
   ```

2. Clone the repository and enter the directory:
   ```bash
   cd feather
   ```

3. Run the test suite:
   ```bash
   noxc test
   ```

## Development Guidelines

- **Zero External Dependencies:** Feather depends solely on the Nox standard library (`nox.db`, `nox.sqlite`, `nox.postgres`, `nox.mysql`, `nox.strings`, `nox.test`).
- **Strict Static Typing:** All functions, methods, fields, and parameters must have explicit static types.
- **Security-First:** Never interpolate raw SQL values. Always use parameter bindings.
- **Test-Driven:** Every feature and bugfix must include unit and integration tests.
- **Dialect Honesty:** Do not claim database features without verifying driver behavior and tests.
- **Code Formatting:** Format code before submitting:
   ```bash
   noxc fmt <file.nox>
   ```

## Pull Request Process

1. Create a feature branch.
2. Add comprehensive tests under `tests/unit/` or `tests/integration/`.
3. Verify all tests pass with `noxc test`.
4. Update `CHANGELOG.md` with your changes.
5. Submit your pull request with a clear description of the problem and solution.
