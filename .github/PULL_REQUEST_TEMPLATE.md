## Description

Please include a summary of the change, along with the motivation and context. Link any related issues using `Fixes #...` or `Closes #...`.

---

## Type of Change

- [ ] 🐛 Bug fix (non-breaking change fixing an issue)
- [ ] ✨ New feature (non-breaking change adding functionality)
- [ ] ⚡ Performance improvement
- [ ] 📝 Documentation update
- [ ] 🛡️ Security hardening
- [ ] 🧪 Test suite addition / improvement

---

## Dialect Support

- [ ] SQLite tested (`:memory:` or shared memory)
- [ ] PostgreSQL compatible
- [ ] MySQL / MariaDB compatible
- [ ] Dialect-independent (AST, codecs, values, session)

---

## Contributor Checklist

- [ ] My code adheres to the project's non-negotiable coding rules in `AGENTS.md`.
- [ ] Every function, parameter, and variable has explicit static types (no untyped code).
- [ ] Zero runtime metaclasses, decorators, or unverified reflection magic.
- [ ] Parameter values are always bound; no string interpolation into SQL values.
- [ ] I have added tests in `tests/unit/` or `tests/integration/` covering my changes.
- [ ] All test suites pass 100% locally with `noxc test`.
- [ ] Static typechecks pass with `noxc check feather.nox` and `noxc check main.nox`.
- [ ] I have updated `CHANGELOG.md` following [Keep a Changelog](https://keepachangelog.com/).
- [ ] Relevant documentation in `docs/` has been updated if public APIs were modified.
