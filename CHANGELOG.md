# Changelog

All notable changes to `aeon-standards` are documented here.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/).

---

## [Unreleased]

---

## [v1.0.4] — 2026-03-16

### Added

- `rerun-downstream.yml`: manually-dispatched workflow that verifies `v1` and
  `v1.X.Y` tags point to the same commit, then reruns the latest failed workflow
  run in a specified downstream repository so consumers can recover from
  transient failures without leaving the platform.

### Changed

- `ci-self-test.yml`: added `rerun-downstream.yml` to the yamllint validation
  step so the new workflow file is syntax-checked on every push and pull request.

---

## [v1.0.3] — 2026-03-16

### Changed

- `python-ci.yml`: added `python-versions` (JSON array string, e.g.
  `'["3.10", "3.11", "3.12"]'`) and `os` inputs so downstream consumers can
  run a version matrix.  The single-version `python-version` input is replaced
  by the array form; the default (`'["3.11"]'`) preserves backward
  compatibility for callers that pass no inputs.
- Job now uses `strategy.matrix` keyed on `python-versions` and `runs-on`
  respects the new `os` input.

---

## [v1.0.2] — 2026-03-16

### Changed

- VERSION bumped to v1.0.2 to re-align `v1` and patch tags with the current
  `main` HEAD after post-#5 doc-churn PRs (#6, #7, #8) advanced `main` without
  triggering a release.  No functional workflow changes; this commit solely
  advances the mutable `v1` pointer to the latest commit.

---

## [v1.0.1] — 2026-03-16

### Changed

- Hardened control-plane release: governed/manual release flow enforced — `v1`
  tag no longer auto-moves on push to `main`; only advances via explicit
  `release-v1.yml` `workflow_dispatch` trigger.
- Action SHAs pinned across all workflow files for supply-chain security.
- Self-test CI (`ci-self-test.yml`) validates reusable workflow YAML on every
  push and pull request.
- `docs/VERSIONING.md` and `CHANGELOG.md` synced to reflect governance policy.

### Fixed

- `release-v1.yml`: renamed `workflow_dispatch` input from `release_note` to
  `annotation` to match the parameter name accepted by the
  `gh workflow run -f` flag.
- `docs/VERSIONING.md`: corrected manual-dispatch parameter name from
  `release_note` to `annotation` to align with the live workflow definition.

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

[Unreleased]: https://github.com/wizardaax/aeon-standards/compare/v1.0.4...HEAD
[v1.0.4]: https://github.com/wizardaax/aeon-standards/compare/v1.0.3...v1.0.4
[v1.0.3]: https://github.com/wizardaax/aeon-standards/compare/v1.0.2...v1.0.3
[v1.0.2]: https://github.com/wizardaax/aeon-standards/compare/v1.0.1...v1.0.2
[v1.0.1]: https://github.com/wizardaax/aeon-standards/compare/v1.0.0...v1.0.1
[v1.0.0]: https://github.com/wizardaax/aeon-standards/releases/tag/v1.0.0
