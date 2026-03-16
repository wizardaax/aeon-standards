# aeon-standards

> **Single source of truth for AEON federated standards: reusable CI/security
> workflows, branch-protection policy patterns, and governance conventions for
> wizardaax repositories.**

---

## Overview

`aeon-standards` is the control-plane repository for federated governance across
all `wizardaax` projects. It provides battle-tested, reusable GitHub Actions
workflows that any downstream repository can call with a single line — ensuring
consistent quality gates, security posture, and coding-style enforcement across
the organisation.

---

## Reusable Workflows

| Workflow | File | Purpose |
|---|---|---|
| **Python CI** | `.github/workflows/python-ci.yml` | Lint, format-check, type-check, and test Python projects |
| **Security** | `.github/workflows/security.yml` | Dependency review, pip-audit, npm audit, secret scanning |

---

## Usage from Consumer Repositories

### Python CI

```yaml
# .github/workflows/ci.yml  (in a downstream repo)
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  python-ci:
    uses: wizardaax/aeon-standards/.github/workflows/python-ci.yml@v1
    # Optional overrides:
    # with:
    #   python-version: "3.12"
    #   working-directory: "src"
```

The workflow runs the following checks in order:

1. **ruff** — fast Python linter
2. **black** — opinionated code formatter (check mode only)
3. **mypy** *(optional, non-fatal)* — static type checker; skipped when no
   mypy configuration is present
4. **pytest** *(conditional)* — test runner; skipped when no test files are
   found; installs `requirements.txt` / `requirements-dev.txt` if present

### Security

```yaml
# .github/workflows/security.yml  (in a downstream repo)
name: Security

on:
  push:
    branches: [main]
  pull_request:

jobs:
  security:
    uses: wizardaax/aeon-standards/.github/workflows/security.yml@v1
    permissions:
      contents: read
      pull-requests: read
```

The workflow runs the following checks in order:

1. **Dependency Review** *(PR-only)* — compares manifest changes against the
   GitHub Advisory Database; fails on `high` severity or above
2. **pip-audit** *(conditional)* — audits Python deps when `requirements*.txt`
   or `pyproject.toml` is present
3. **npm audit** *(conditional)* — audits Node.js deps when `package-lock.json`
   is present
4. **gitleaks** — scans full commit history for accidentally committed secrets

---

## Branch Protection Integration

Both workflows expose **stable job names** (`python-ci` and `security`) that
you can reference directly in branch-protection required-status-check rules:

| Check name | Workflow |
|---|---|
| `python-ci` | `python-ci.yml` |
| `security` | `security.yml` |

---

## Version Pinning

Always pin to a **major version tag** (e.g. `@v1`) rather than `@main`.  
Major version tags are mutable references that receive backwards-compatible
updates; pinning to `@main` may expose you to breaking changes.

```yaml
uses: wizardaax/aeon-standards/.github/workflows/python-ci.yml@v1
```

For maximum reproducibility in highly-regulated environments you may pin to a
specific commit SHA instead:

```yaml
uses: wizardaax/aeon-standards/.github/workflows/python-ci.yml@<full-sha>
```

---

## Compatibility Policy

- **Minor / patch releases** within `v1` are backwards-compatible; the `@v1`
  tag is updated to point to the latest compatible release.
- **Breaking changes** (new required inputs, removed steps, renamed jobs) will
  always result in a new major version (`v2`, `v3`, …).
- Deprecated major versions receive security fixes only and are announced in the
  [CHANGELOG](./CHANGELOG.md) before removal.

See [docs/VERSIONING.md](./docs/VERSIONING.md) for the full versioning policy.

---

## Repository Structure

```
aeon-standards/
├── .github/
│   └── workflows/
│       ├── python-ci.yml          # Reusable Python CI workflow
│       ├── security.yml           # Reusable Security workflow
│       ├── release-v1.yml         # Governed release tagging (workflow_dispatch)
│       ├── ci-self-test.yml       # Self-test: YAML lint on push/PR
│       └── rerun-downstream.yml   # Verify tags and rerun failed downstream jobs
├── docs/
│   └── VERSIONING.md              # Versioning and release policy
├── CHANGELOG.md
├── LICENSE
└── README.md
```

---

## Contributing

1. Open a pull request against `main`.
2. Use [Conventional Commits](https://www.conventionalcommits.org/) for all
   commit messages and PR titles (e.g. `feat:`, `fix:`, `docs:`, `chore:`).
3. All changes must pass the repository's own CI before merge.
4. Breaking workflow changes **must** bump the major version; document the
   change in `docs/VERSIONING.md`.

---

## License

[MIT](./LICENSE)
