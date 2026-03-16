# Versioning Policy

This document defines how `aeon-standards` workflow versions are maintained,
when to cut a new major version, and the support lifecycle for each version.

---

## Versioning Scheme

`aeon-standards` uses **Semantic Versioning 2.0.0** adapted for GitHub Actions
reusable workflows, where the public API is the set of workflow inputs, outputs,
job names, and required secrets.

```
vMAJOR.MINOR.PATCH
```

| Segment | Changes that trigger a bump |
|---|---|
| `MAJOR` | Backwards-incompatible changes (see below) |
| `MINOR` | New optional inputs, new non-breaking steps, new workflow files |
| `PATCH` | Bug fixes, dependency pin updates, documentation corrections |

---

## Major Version Tags (e.g. `v1`)

Each major version is backed by a **mutable Git tag** (e.g. `v1`) that is
updated to the latest backwards-compatible release within that major line.
Consumers pin to `@v1` and automatically receive minor/patch improvements
without changing their workflow files.

```
v1  ──► v1.0.0 ──► v1.1.0 ──► v1.1.1  (tag moves forward)
v2  ──► v2.0.0 ──► …              (new tag, separate line)
```

Immutable tags (e.g. `v1.2.3`) are also pushed for consumers who require
reproducible, pinned references.

---

## What Is — and Is Not — a Breaking Change

### Breaking (requires `v2`)

- Removing or renaming a workflow file referenced by consumers
- Removing or renaming a `workflow_call` input or secret
- Changing the **default value** of an existing input in a way that alters
  observable behaviour
- Renaming a job (breaks branch-protection status-check names)
- Adding a **required** input (consumers without `with:` block would break)
- Changing the failure semantics of an existing step (e.g. making a previously
  non-fatal step fatal)

### Non-breaking (stays within `v1`)

- Adding a new **optional** input with a sensible default
- Adding, removing, or reordering steps that do not change the job's pass/fail
  outcome for previously passing consumers
- Updating pinned action versions (e.g. `actions/checkout@v3` → `@v4`)
- Performance improvements, parallelism changes, log formatting
- New workflow files that do not affect existing ones

---

## How `v1` Is Maintained

1. All changes land on `main` via pull requests using
   [Conventional Commits](https://www.conventionalcommits.org/).
2. After merging a non-breaking change, a maintainer runs the
   **Release v1** workflow (`release-v1.yml`) via the GitHub Actions
   **"Run workflow"** button, supplying:
   - `patch_version` — the immutable tag to create (e.g. `v1.0.1`)
   - `release_note` — a one-line annotation (e.g. `fix: pin action SHAs`)

   The workflow automatically:
   - Creates the immutable patch tag (e.g. `v1.0.1`)
   - Force-updates the mutable `v1` tag to the same commit
3. The [CHANGELOG](../CHANGELOG.md) entry is updated in the PR before merging.

> **Governance note:** The `v1` tag is **never** moved automatically on push
> to `main`.  It only advances when a maintainer explicitly triggers the
> release workflow.  This prevents accidental breakage of downstream consumers.

---

## When to Cut `v2`

A new major version is cut when a **breaking change** (see above) is necessary.
The process is:

1. Open a PR titled `feat!: <description>` (the `!` signals breaking change).
2. Update this document to describe the new major version.
3. After merge, tag `v2.0.0` and create the mutable `v2` tag.
4. Announce the new version and migration guide in the PR description and
   CHANGELOG.

---

## Support Lifecycle

| Version | Status | Notes |
|---|---|---|
| `v1` | **Active** | Receives features, fixes, and security patches |
| `v0` | End-of-life | No longer supported |

- A major version is declared **deprecated** when its successor has been stable
  for at least **30 days**.
- A deprecated version continues to receive **security fixes only** for a
  further **90 days**, after which it is declared end-of-life and the mutable
  tag is frozen.
- Consumers are notified of deprecations via a pinned issue in this repository
  and a note in the CHANGELOG.

---

## First Release (`v1.0.0`)

The repository is ready for tagging as `v1.0.0` / `v1` once the following
conditions are met:

- [x] `.github/workflows/python-ci.yml` exists and is callable
- [x] `.github/workflows/security.yml` exists and is callable
- [x] `README.md` documents consumer usage
- [x] This `docs/VERSIONING.md` document exists
- [x] PR is merged into `main`
- [x] Maintainer pushes `v1.0.0` and `v1` tags via Release v1 workflow
