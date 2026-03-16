# Changelog

All notable changes to `aeon-standards` are documented here.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/).

---

## [Unreleased]

_Changes merged to `main` that have not yet been tagged as a release._

---

## [v1.0.0] — 2026-03-16

### Added

- `python-ci.yml` — reusable workflow: ruff lint, black format-check, mypy
  (optional), pytest (conditional on test-file presence).
- `security.yml` — reusable workflow: GitHub Dependency Review (PR-only),
  pip-audit (conditional), npm audit (conditional), gitleaks secret scan.
- `release-v1.yml` — governance-gated release workflow; creates an immutable
  `v1.X.Y` patch tag and force-updates the mutable `v1` major tag only when
  explicitly triggered via `workflow_dispatch`.
- `ci-self-test.yml` — repository self-test: yamllint validation of all
  reusable workflow files on every push/PR.
- `docs/VERSIONING.md` — versioning and release-lifecycle policy.
- `README.md` — consumer usage guide with branch-protection integration notes.
- Action SHAs pinned for supply-chain security in all workflow files.

### Notes

- Stable job names (`python-ci`, `security`) are suitable for branch-protection
  required-status-check rules in downstream repositories.
- Consumer repos can reference workflows via:
  ```yaml
  uses: wizardaax/aeon-standards/.github/workflows/python-ci.yml@v1
  uses: wizardaax/aeon-standards/.github/workflows/security.yml@v1
  ```

[Unreleased]: https://github.com/wizardaax/aeon-standards/compare/v1.0.0...HEAD
[v1.0.0]: https://github.com/wizardaax/aeon-standards/releases/tag/v1.0.0
