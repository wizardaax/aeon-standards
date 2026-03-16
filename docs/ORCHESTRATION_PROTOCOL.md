# Orchestration Protocol

This document describes the **GitHub-native orchestrator framework** built into
`wizardaax/aeon-standards`. It uses only GitHub primitives — Issues, Labels,
Comments, Actions, and `repository_dispatch` — to run multi-agent orchestration
with no external infrastructure.

---

## Architecture overview

```text
Operator
  │
  ▼  gh workflow run orchestrator-dispatch.yml
┌──────────────────────────────────────────────────────────┐
│  orchestrator-dispatch.yml                               │
│  • Generates run ID                                      │
│  • Creates run tracking issue (label: orchestrator:run)  │
│  • Posts task envelopes as issue comments                │
│  • Fires repository_dispatch for independent tasks       │
└──────────────────┬───────────────────────────────────────┘
                   │ repository_dispatch
         ┌─────────┴──────────┐
         ▼                    ▼
   agent-ci.yml     agent-federation.yml
   (task-002)           (task-001)
         │                    │
         │ result envelope    │ result envelope
         │ (issue comment)    │ (issue comment)
         │                    │
         └─────────┬──────────┘
                   │ repository_dispatch
                   ▼  orchestrator.reduce
         orchestrator-reducer.yml
         • Parses all envelopes
         • Dispatches unblocked tasks
         • On all done → posts final summary
                   │
                   ▼ repository_dispatch  agent.task.auditor
             agent-auditor.yml
             (task-003, depends on task-001 + task-002)
                   │ result envelope
                   ▼
         orchestrator-reducer.yml
         • Final summary + issue close
```

The **orchestrator-scheduler** runs on a cron schedule (every 10 minutes) to
detect and retry stale tasks that exceed their SLA budget.

---

## Run state machine

```
NEW → PLANNED → DISPATCHED → RUNNING → REDUCING → DONE
                                                 ↘ FAILED
                                                 ↘ ESCALATED
```

Escalation conditions (task result `escalate: true`):

- Federation drift detected in `safe` mode
- Auditor findings at or above `audit_block_severity`
- Task exceeds `max_attempts` retries

---

## Message bus protocol

### Transport

| Layer | Mechanism |
|-------|-----------|
| Primary state store | JSON envelopes in issue comments |
| Async trigger | `repository_dispatch` events |
| State lock | GitHub issue labels |

### Envelope format

Each task is tracked by a **task envelope** — a strict JSON object posted as a
fenced code block in an issue comment.  Comments are marked with an HTML anchor
so the reducer can identify them:

```text
<!-- orchestrator:envelope task_id="task-001" run_id="aeon-2026-..." -->
```json
{
  "run_id":           "aeon-2026-03-16T22-14-55Z-7f3a1b2c",
  "goal":             "Propagate python-ci@v1.0.4 across fleet",
  "repo_scope":       ["wizardaax/recursive-field-math-pro","wizardaax/SCE-88"],
  "task_id":          "task-001",
  "agent":            "federation",
  "state":            "done",
  "attempt":          1,
  "depends_on":       [],
  "artifacts":        [{"name":"alignment-report","url":"...","type":"report"}],
  "result":           {"ok":true,"summary":"All repos aligned to v1.0.4"},
  "timestamp":        "2026-03-16T22:18:02Z",
  "run_issue_number": 42,
  "mode":             "safe"
}
```
```

The envelope schema is formally defined in [`schemas/envelope.json`](../schemas/envelope.json).

### Labels

| Label | Meaning |
|-------|---------|
| `orchestrator:run` | Issue tracks an active orchestrator run |
| `state:queued` | Task is queued |
| `state:running` | Task is executing |
| `state:blocked` | Task is blocked (SLA breach / failed dependency) |
| `state:done` | Task completed successfully |
| `state:failed` | Task failed permanently |
| `priority:p1` | Critical / SLA breach |
| `priority:p2` | Normal (default) |
| `priority:p3` | Low urgency / background |
| `agent:builder` | Builder agent task |
| `agent:auditor` | Auditor agent task |
| `agent:ci` | CI agent task |
| `agent:federation` | Federation agent task |
| `human-gate:pending` | Awaiting `/approve` operator command |
| `human-gate:approved` | Operator has approved merge gate |

---

## Agents

### Builder (`agent-builder.yml`)

| | |
|-|-|
| Trigger | `repository_dispatch` `event_type=agent.task.builder` |
| Input | `goal`, `repo_scope`, `mode` |
| Output | PR URL(s), changed files, commit summary |
| Fail condition | No PR opened and no documented reason |

Reads the policy denylist from `orchestrator-policy.yml`.
In `safe` mode opens **draft** PRs; in `aggressive` mode opens **ready** PRs.

### Auditor (`agent-auditor.yml`)

| | |
|-|-|
| Trigger | `repository_dispatch` `event_type=agent.task.auditor` |
| Verifies | lint/test/security CI status, branch protection check names, action SHA pinning |
| Output | `PASS` or `FAIL` + required remediations |
| Block threshold | `audit_block_severity` from `orchestrator-policy.yml` (default: `high`) |

### CI Agent (`agent-ci.yml`)

| | |
|-|-|
| Trigger | `repository_dispatch` `event_type=agent.task.ci` |
| Action | Finds latest failing run per repo, fires `gh run rerun --failed` |
| Output | Pass-rate delta, residual blocker list |

### Federation Agent (`agent-federation.yml`)

| | |
|-|-|
| Trigger | `repository_dispatch` `event_type=agent.task.federation` |
| Checks | Reusable workflow refs (`@v1`, `@v1.x.y`) vs. control-plane `VERSION` |
| Drift in `safe` mode | Reports drift, sets `escalate: true` |
| Drift in `aggressive` mode | Opens patch PRs up to `max_auto_prs` |

---

## Orchestrator workflows

### `orchestrator-dispatch.yml`

**Trigger:** `workflow_dispatch` with `goal`, `repo_scope`, `mode`

1. Generates `run_id` (format: `aeon-YYYY-MM-DDTHH-MM-SSZ-<8hex>`).
2. Creates an issue with label `orchestrator:run,state:running`.
3. Posts task envelopes (state=`queued`) as issue comments.
4. Fires `repository_dispatch` for independent tasks (no `depends_on`).

### `orchestrator-reducer.yml`

**Triggers:** `repository_dispatch` `orchestrator.reduce` OR `issue_comment` created

1. Resolves the run context (run ID + issue number).
2. Handles operator commands (`/approve <run_id>`, `/abort <run_id>`).
3. Parses all `orchestrator:envelope` comments.
4. Dispatches newly unblocked tasks.
5. When all tasks resolve: posts final summary, updates labels, closes issue.

### `orchestrator-scheduler.yml`

**Trigger:** cron `*/10 * * * *` + `workflow_dispatch`

1. Loads SLA and retry budget from `orchestrator-policy.yml`.
2. Lists open issues with `orchestrator:run` + `state:running`.
3. For each stale task (age > `task_sla_minutes`):
   - If `attempt < max_attempts`: re-dispatches.
   - If `attempt >= max_attempts`: marks `blocked`, posts warning.
4. At `00:00 UTC`: posts daily health-KPI digest issue.

---

## Idempotency

Every task is identified by the composite key `run_id + task_id + attempt`.
The reducer always uses the **highest-attempt** envelope for each `task_id`.
Re-dispatching an already-completed task is safe: the agent will post a new
envelope, but the reducer will ignore it if the previous attempt already
reached `done`.

---

## Security model

All of the following are enforced by `orchestrator-policy.yml`:

| Control | Details |
|---------|---------|
| Token | `GH_ORCHESTRATOR_TOKEN` (fine-grained PAT or GitHub App) |
| File denylist | Agents cannot edit `.github/workflows/release*`, `orchestrator-policy.yml`, `CODEOWNERS`, etc. |
| Authorized repos | Agents may only act on repos listed in `authorized_repos` |
| Action SHA pinning | All third-party actions must use full commit-SHA refs |
| Human gate | `require_human_gate: true` blocks auto-merge until `/approve` |
| Audit severity | Findings at `audit_block_severity` or above block the run |

CODEOWNERS review is required for any change to `orchestrator-policy.yml` or
workflow files.

---

## Operator quick-start

### Trigger a run

```bash
gh workflow run orchestrator-dispatch.yml \
  -R wizardaax/aeon-standards \
  -f goal="Propagate python-ci@v1.0.4 and rerun failed checks across fleet" \
  -f repo_scope='["wizardaax/recursive-field-math-pro","wizardaax/SCE-88","wizardaax/wizardaax.github.io"]' \
  -f mode="safe"
```

### Check active runs

```bash
gh issue list -R wizardaax/aeon-standards \
  -l "orchestrator:run" -l "state:running"
```

### Approve a merge gate

```bash
gh issue comment <run_issue_number> \
  -R wizardaax/aeon-standards \
  --body "/approve aeon-2026-03-16T22-14-55Z-7f3a1b2c"
```

### Abort a run

```bash
gh issue comment <run_issue_number> \
  -R wizardaax/aeon-standards \
  --body "/abort aeon-2026-03-16T22-14-55Z-7f3a1b2c"
```

### Force-check for stale tasks now

```bash
gh workflow run orchestrator-scheduler.yml \
  -R wizardaax/aeon-standards \
  -f dry_run=true
```

---

## MVP acceptance criteria

- [x] One run issue tracks the full task DAG.
- [x] Builder creates at least one draft PR automatically.
- [x] Auditor blocks merge on policy failures (`escalate: true`).
- [x] CI agent reruns failed jobs and reports pass-rate delta.
- [x] Reducer posts a deterministic final summary.
- [x] Full run is reproducible from the issue timeline and artifacts.

---

## Schemas

| File | Description |
|------|-------------|
| [`schemas/envelope.json`](../schemas/envelope.json) | JSON Schema for task envelopes |
| [`schemas/task.json`](../schemas/task.json) | JSON Schema for task definitions |
| [`schemas/result.json`](../schemas/result.json) | JSON Schema for task results |
| [`orchestrator-policy.yml`](../orchestrator-policy.yml) | Policy-as-code configuration |
