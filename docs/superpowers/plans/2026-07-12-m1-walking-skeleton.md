# M1 Walking Skeleton Implementation Plan (tnf-auto-reviewer)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** The thinnest end-to-end unattended PR review: a podman container (or local session) takes a TNF PR URL, runs intake → code-review harvest → vet → compose over `review-state.yaml`, holds the composed review (`hold` mode default), and a later ad-hoc session posts it verbatim to GitHub.

**Architecture:** Every stage is a plugin skill that reads/writes only `<state_dir>/review-state.yaml` (the bus). A Claude Code **dynamic workflow** (`.claude/workflows/review.js`) sequences the stages as fresh subagents and skips stages already `done` (idempotent restart). A Containerfile bakes blobless TNF repo clones + vendored domain context + this plugin; the entrypoint runs `claude -p "/tnf-review …"` headless. Spec: `repos/tnf-auto-reviewer/docs/design/2026-07-12-poc-design.md`.

**Tech Stack:** Claude Code plugin (SKILL.md skills, dynamic workflow JS), YAML state validated by JSON Schema (python3 + pyyaml + jsonschema + pytest), `gh` CLI for all GitHub API calls, podman + Fedora base image.

## Global Constraints

- Claude Code **≥ v2.1.154** required (dynamic workflows); container installs latest CLI.
- Dynamic workflow scripts have **no filesystem or shell access** — every disk/network touch happens inside an `agent()` call.
- The pipeline **never submits an APPROVE review** — only `COMMENT` or `REQUEST_CHANGES` (spec: "Agent ever approving a PR. Never.").
- **Never post a partial review**: a failed stage sets `status: failed` in state and the run stops with a thrown error / non-zero exit.
- `posting_mode` default is **`hold`**; `auto` is opt-in via workflow arg / `POSTING_MODE` env.
- PoC hardcodes TNF (repo list, context); the plugin is **self-contained** — no workspace-plugin dependency.
- Credentials (`GITHUB_TOKEN`, `ANTHROPIC_API_KEY`, `JIRA_PAT`) are **never baked** into the image — env-injected only.
- Every stage validates `review-state.yaml` against the schema before declaring itself done: `python3 "${CLAUDE_PLUGIN_ROOT}/scripts/validate_state.py" <state-file>` must print `OK`.
- Stage-status enum everywhere: `pending | running | done | skipped | failed | held`.
- Timestamps: UTC ISO-8601 (`date -u +%Y-%m-%dT%H:%M:%SZ`).
- All work happens in the git worktree `repos/tnf-auto-reviewer/.worktrees/m1-skeleton` (branch `m1-skeleton` off `main`); commit at the end of every task; do **not** push or open a PR until the final task's checklist says so.

## File Structure

All paths relative to the worktree root (`/Users/pfontani/Workspace/tnf-dev-env/repos/tnf-auto-reviewer/.worktrees/m1-skeleton`):

```
.claude/workflows/review.js          NEW  dynamic-workflow orchestrator (/tnf-review)
Containerfile                        NEW  runtime image: baked repos + context + plugin
entrypoint.sh                        NEW  container entrypoint (fetch, slug, claude -p)
context/tnf/                         NEW  vendored TNF domain context (context.md, architecture.md, debugging.md)
docs/design/review-state.schema.yaml REWRITE  formal JSON Schema (draft 2020-12, in YAML)
scripts/validate_state.py            NEW  schema validator (library + CLI)
skills/intake/SKILL.md               NEW  stage 1: env + seed state
skills/harvest/SKILL.md              NEW  stage 2 (state key `code-review`): /code-review → findings
skills/vet/SKILL.md                  NEW  stage 3: adversarial vet, round 1
skills/compose/SKILL.md              NEW  stage 4: composed review + findings.md
skills/post/SKILL.md                 NEW  stage 5: post verbatim from state (also the hold-resume entry)
skills/run/SKILL.md                  REWRITE  thin interactive entrypoint
tests/fixtures/valid-state.yaml      NEW  full mid-pipeline instance (schema tests)
tests/fixtures/invalid-state.yaml    NEW  bad enums / missing fields (schema tests)
tests/fixtures/code-review-output.md NEW  captured /code-review output (pins harvest coupling)
tests/test_schema.py                 NEW  pytest: validator behavior
README.md                            MODIFY  status: M1 implemented
```

Stage-key ↔ skill mapping (the state file's `stages:` keys never change; M2 fills the three `skipped` ones):

| stages: key | skill | M1 |
|---|---|---|
| `intake` | `tnf-reviewer:intake` | yes |
| `code-review` | `tnf-reviewer:harvest` | yes |
| `vet` | `tnf-reviewer:vet` | yes |
| `spec-compliance` / `cross-repo` / `arch-fit` | (M2) | seeded `skipped` |
| `compose` | `tnf-reviewer:compose` | yes |
| `post` | `tnf-reviewer:post` | yes (`held` in hold mode) |

**Shared eval directory:** stage tasks (2–6) verify against a *replayed merged TNF PR* (the spec's pre-flight). Task 2 creates `~/tnf-reviews-eval/pr-<repo>-<n>` from a real merged PR; tasks 3–6 continue on that same directory. This sequential chain is intentional — it is the spec's stage-test story ("every stage skill runnable standalone against a fixture state dir").

---

### Task 1: Worktree, state schema, validator, and schema tests

**Files:**
- Create: `repos/tnf-auto-reviewer/.worktrees/m1-skeleton/` (worktree, branch `m1-skeleton`)
- Rewrite: `docs/design/review-state.schema.yaml`
- Create: `scripts/validate_state.py`
- Create: `tests/fixtures/valid-state.yaml`, `tests/fixtures/invalid-state.yaml`
- Create: `tests/test_schema.py`

**Interfaces:**
- Consumes: nothing (first task).
- Produces: the state contract every later task writes to — field names exactly as in the schema below; `validate(path) -> list[jsonschema.ValidationError]` and CLI `python3 scripts/validate_state.py <file>` printing `OK` (exit 0) or errors (exit 1).

- [ ] **Step 1: Create the worktree**

```bash
git -C /Users/pfontani/Workspace/tnf-dev-env/repos/tnf-auto-reviewer worktree add .worktrees/m1-skeleton -b m1-skeleton main
cd /Users/pfontani/Workspace/tnf-dev-env/repos/tnf-auto-reviewer/.worktrees/m1-skeleton
```

Expected: `Preparing worktree (new branch 'm1-skeleton')`. All later steps run from this directory.

- [ ] **Step 2: Ensure python deps**

```bash
python3 -c "import yaml, jsonschema, pytest" 2>/dev/null || python3 -m pip install --user pyyaml jsonschema pytest
```

- [ ] **Step 3: Write the failing tests**

Create `tests/test_schema.py`:

```python
"""Schema tests: review-state.yaml instances validate against the JSON Schema."""
import pathlib
import subprocess
import sys

REPO = pathlib.Path(__file__).resolve().parents[1]
sys.path.insert(0, str(REPO / "scripts"))
from validate_state import validate  # noqa: E402

FIXTURES = pathlib.Path(__file__).parent / "fixtures"


def test_valid_instance_passes():
    assert validate(FIXTURES / "valid-state.yaml") == []


def test_invalid_instance_reports_errors():
    messages = " | ".join(e.message for e in validate(FIXTURES / "invalid-state.yaml"))
    assert "'reviewing'" in messages          # bad stage-status enum value
    assert "'maybe'" in messages              # bad posting_mode enum value
    assert "'severity' is a required property" in messages


def test_cli_ok_and_failure_exit_codes():
    ok = subprocess.run(
        [sys.executable, str(REPO / "scripts/validate_state.py"), str(FIXTURES / "valid-state.yaml")],
        capture_output=True, text=True,
    )
    assert ok.returncode == 0 and ok.stdout.strip() == "OK"
    bad = subprocess.run(
        [sys.executable, str(REPO / "scripts/validate_state.py"), str(FIXTURES / "invalid-state.yaml")],
        capture_output=True, text=True,
    )
    assert bad.returncode == 1 and bad.stdout.strip()
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_schema.py -v`
Expected: FAIL/ERROR with `ModuleNotFoundError: No module named 'validate_state'` (nothing exists yet).

- [ ] **Step 5: Write the schema**

Replace the entire contents of `docs/design/review-state.schema.yaml` (this **finalizes** the reconstructed schema — spec's M1 deliverable; design deltas applied: `posting_mode`, `held` status, `post` split from `compose`, `human-gate` removed, `environment` + composed-review bodies added so the post stage is verbatim):

```yaml
# review-state.yaml — JSON Schema (draft 2020-12, authored in YAML).
# THE pipeline bus: every stage reads/writes only this file, so each stage is
# a fresh, resumable subagent. Finalized in M1 from the reconstructed draft.
# Validate instances with: python3 scripts/validate_state.py <file>
$schema: "https://json-schema.org/draft/2020-12/schema"
title: review-state
type: object
additionalProperties: false
required: [schema_version, pr, head_sha, reviewed_sha, previous_shas, posting_mode,
           created_at, updated_at, repos_in_scope, environment, stages, findings, review]
$defs:
  nullable_string:
    oneOf: [{type: string}, {type: "null"}]
  timestamp:
    type: string
    format: date-time
  nullable_timestamp:
    oneOf: [{$ref: "#/$defs/timestamp"}, {type: "null"}]
  stage:
    type: object
    additionalProperties: false
    required: [status]
    properties:
      status: {enum: [pending, running, done, skipped, failed, held]}
      started_at: {$ref: "#/$defs/nullable_timestamp"}
      finished_at: {$ref: "#/$defs/nullable_timestamp"}
      notes: {$ref: "#/$defs/nullable_string"}
  finding:
    type: object
    additionalProperties: false
    required: [id, source, severity, title, detail, anchor, vet, lifecycle, posting]
    properties:
      id: {type: string, pattern: "^F-[0-9]{3,}$"}   # stable, never reused
      source: {enum: [code-review, spec-compliance, cross-repo, arch-fit]}
      category: {enum: [bug, security, contract-break, spec-gap, pattern, clarity, minor]}
      severity: {enum: [critical, major, minor, clarity]}
      title: {type: string}
      detail: {type: string}                          # full body, markdown
      anchor:
        type: object
        additionalProperties: false
        required: [file, line, sha]
        properties:
          file: {type: string}                        # repo-relative
          line: {type: integer}
          side: {enum: [LEFT, RIGHT]}
          sha: {type: string}                         # head this anchor was computed against
          diff_position: {oneOf: [{type: integer}, {type: "null"}]}
      vet:
        type: object
        additionalProperties: false
        required: [verdict, rationale, vetted_at, round]
        properties:
          verdict: {enum: [pending, confirmed, dropped]}
          rationale: {$ref: "#/$defs/nullable_string"}
          vetted_at: {$ref: "#/$defs/nullable_timestamp"}
          round: {type: integer}                      # 1 = harvest, 2 = context passes (M2)
      lifecycle: {enum: [active, stale, resolved]}    # delta re-review (M3)
      posting:
        type: object
        additionalProperties: false
        required: [decision, comment_body, comment_id, posted_at]
        properties:
          decision: {enum: [pending, inline, summary, suppressed]}
          comment_body: {$ref: "#/$defs/nullable_string"}   # exact text posted inline; ends with <!-- tnf-reviewer:F-NNN -->
          comment_id: {$ref: "#/$defs/nullable_string"}     # idempotency key: set => NEVER repost
          posted_at: {$ref: "#/$defs/nullable_timestamp"}
      precedents: {type: array, items: {type: string}}      # arch-fit (M2)
      consumers: {type: array, items: {type: string}}       # cross-repo (M2)
properties:
  schema_version: {const: 1}
  pr:
    type: object
    additionalProperties: false
    required: [url, repo, number, base_ref, author]
    properties:
      url: {type: string}
      repo: {type: string}                            # "owner/name"
      number: {type: integer}
      base_ref: {type: string}
      author: {type: string}
      title: {type: string}
  head_sha: {type: string}
  reviewed_sha: {$ref: "#/$defs/nullable_string"}     # last head fully piped (delta, M3)
  previous_shas: {type: array, items: {type: string}}
  posting_mode: {enum: [auto, hold]}                  # workflow input param, recorded for audit
  created_at: {$ref: "#/$defs/timestamp"}
  updated_at: {$ref: "#/$defs/timestamp"}
  repos_in_scope: {type: array, items: {type: string}}
  tracker:                                            # intent.md snapshot (M2)
    type: object
    additionalProperties: false
    properties:
      type: {enum: [jira, none]}
      id: {$ref: "#/$defs/nullable_string"}
      url: {$ref: "#/$defs/nullable_string"}
  environment:
    type: object
    additionalProperties: false
    required: [repos_root, worktree_path]
    properties:
      repos_root: {type: string}
      worktree_path: {type: string}
      context_root: {$ref: "#/$defs/nullable_string"}
  stages:
    type: object
    additionalProperties: false
    required: [intake, code-review, vet, spec-compliance, cross-repo, arch-fit, compose, post]
    properties:
      intake: {$ref: "#/$defs/stage"}
      code-review: {$ref: "#/$defs/stage"}
      vet: {$ref: "#/$defs/stage"}
      spec-compliance: {$ref: "#/$defs/stage"}        # M2 — seeded `skipped` in M1
      cross-repo: {$ref: "#/$defs/stage"}             # M2 — seeded `skipped` in M1
      arch-fit: {$ref: "#/$defs/stage"}               # M2 — seeded `skipped` in M1
      compose: {$ref: "#/$defs/stage"}
      post: {$ref: "#/$defs/stage"}                   # `held` = composed, awaiting human resume
  findings:
    type: array
    items: {$ref: "#/$defs/finding"}
  review:
    type: object
    additionalProperties: false
    required: [verdict_recommendation, body, summary_comment_id, submitted, submitted_at]
    properties:
      verdict_recommendation:
        oneOf: [{enum: [REQUEST_CHANGES, COMMENT]}, {type: "null"}]   # NEVER APPROVE
      body: {$ref: "#/$defs/nullable_string"}         # exact top-level review body posted
      summary_comment_id: {$ref: "#/$defs/nullable_string"}
      submitted: {type: boolean}
      submitted_at: {$ref: "#/$defs/nullable_timestamp"}
```

- [ ] **Step 6: Write the validator**

Create `scripts/validate_state.py`:

```python
#!/usr/bin/env python3
"""Validate a review-state.yaml instance against docs/design/review-state.schema.yaml.

Library: validate(path) -> list of jsonschema.ValidationError (empty = valid).
CLI:     validate_state.py <review-state.yaml>  -> prints OK (exit 0) or errors (exit 1).
"""
import pathlib
import sys

import jsonschema
import yaml

SCHEMA_PATH = pathlib.Path(__file__).resolve().parents[1] / "docs" / "design" / "review-state.schema.yaml"


def validate(instance_path):
    schema = yaml.safe_load(SCHEMA_PATH.read_text())
    instance = yaml.safe_load(pathlib.Path(instance_path).read_text())
    validator = jsonschema.Draft202012Validator(schema)
    return sorted(validator.iter_errors(instance), key=lambda e: [str(p) for p in e.path])


def main(argv):
    if len(argv) != 2:
        print("usage: validate_state.py <review-state.yaml>")
        return 2
    errors = validate(argv[1])
    if errors:
        for err in errors:
            where = "/".join(str(p) for p in err.path) or "<root>"
            print(f"{where}: {err.message}")
        return 1
    print("OK")
    return 0


if __name__ == "__main__":
    sys.exit(main(sys.argv))
```

- [ ] **Step 7: Write the fixtures**

Create `tests/fixtures/valid-state.yaml` (a mid-pipeline instance — vet done, compose pending):

```yaml
schema_version: 1
pr:
  url: https://github.com/openshift/cluster-etcd-operator/pull/1620
  repo: openshift/cluster-etcd-operator
  number: 1620
  base_ref: main
  author: someuser
  title: "TNF: guard fencing handover against nil status"
head_sha: a1b2c3d4e5f60718293a4b5c6d7e8f9012345678
reviewed_sha: null
previous_shas: []
posting_mode: hold
created_at: 2026-07-12T10:00:00Z
updated_at: 2026-07-12T10:42:00Z
repos_in_scope: [cluster-etcd-operator, machine-config-operator]
tracker: {type: none, id: null, url: null}
environment:
  repos_root: /repos
  worktree_path: /state/pr-cluster-etcd-operator-1620/worktree
  context_root: /context
stages:
  intake:          {status: done, started_at: 2026-07-12T10:00:00Z, finished_at: 2026-07-12T10:02:00Z, notes: null}
  code-review:     {status: done, started_at: 2026-07-12T10:02:10Z, finished_at: 2026-07-12T10:20:00Z, notes: "harvested 2 findings"}
  vet:             {status: done, started_at: 2026-07-12T10:20:05Z, finished_at: 2026-07-12T10:40:00Z, notes: "confirmed 1, dropped 1"}
  spec-compliance: {status: skipped, started_at: null, finished_at: null, notes: "M2"}
  cross-repo:      {status: skipped, started_at: null, finished_at: null, notes: "M2"}
  arch-fit:        {status: skipped, started_at: null, finished_at: null, notes: "M2"}
  compose:         {status: pending, started_at: null, finished_at: null, notes: null}
  post:            {status: pending, started_at: null, finished_at: null, notes: null}
findings:
  - id: F-001
    source: code-review
    category: bug
    severity: major
    title: Nil pointer when fencing status is unset during handover
    detail: |
      `ensureHandover` dereferences `status.Fencing.Mode` before the TNF
      controller has populated it; a restart mid-handover panics the operator.
    anchor: {file: pkg/tnf/handover.go, line: 214, side: RIGHT, sha: a1b2c3d4e5f60718293a4b5c6d7e8f9012345678, diff_position: null}
    vet: {verdict: confirmed, rationale: "Reproduced by reading pkg/tnf/handover.go:210-220; no nil guard on the path.", vetted_at: 2026-07-12T10:35:00Z, round: 1}
    lifecycle: active
    posting: {decision: pending, comment_body: null, comment_id: null, posted_at: null}
  - id: F-002
    source: code-review
    category: clarity
    severity: clarity
    title: Misleading comment on retry loop
    detail: Comment says exponential backoff but the loop is fixed-interval.
    anchor: {file: pkg/tnf/handover.go, line: 88, side: RIGHT, sha: a1b2c3d4e5f60718293a4b5c6d7e8f9012345678, diff_position: null}
    vet: {verdict: dropped, rationale: "Comment matches after the follow-up hunk at line 95; not misleading in context.", vetted_at: 2026-07-12T10:38:00Z, round: 1}
    lifecycle: active
    posting: {decision: pending, comment_body: null, comment_id: null, posted_at: null}
review:
  verdict_recommendation: null
  body: null
  summary_comment_id: null
  submitted: false
  submitted_at: null
```

Create `tests/fixtures/invalid-state.yaml` (three deliberate violations: bad stage status enum, bad posting_mode enum, finding missing `severity`):

```yaml
schema_version: 1
pr:
  url: https://github.com/openshift/cluster-etcd-operator/pull/1620
  repo: openshift/cluster-etcd-operator
  number: 1620
  base_ref: main
  author: someuser
head_sha: a1b2c3d4e5f60718293a4b5c6d7e8f9012345678
reviewed_sha: null
previous_shas: []
posting_mode: maybe
created_at: 2026-07-12T10:00:00Z
updated_at: 2026-07-12T10:42:00Z
repos_in_scope: []
environment:
  repos_root: /repos
  worktree_path: /state/pr-cluster-etcd-operator-1620/worktree
stages:
  intake:          {status: done}
  code-review:     {status: reviewing}
  vet:             {status: pending}
  spec-compliance: {status: skipped}
  cross-repo:      {status: skipped}
  arch-fit:        {status: skipped}
  compose:         {status: pending}
  post:            {status: pending}
findings:
  - id: F-001
    source: code-review
    title: Missing severity on purpose
    detail: This finding omits the required severity field.
    anchor: {file: pkg/tnf/handover.go, line: 214, sha: a1b2c3d4e5f60718293a4b5c6d7e8f9012345678}
    vet: {verdict: pending, rationale: null, vetted_at: null, round: 1}
    lifecycle: active
    posting: {decision: pending, comment_body: null, comment_id: null, posted_at: null}
review:
  verdict_recommendation: null
  body: null
  summary_comment_id: null
  submitted: false
  submitted_at: null
```

- [ ] **Step 8: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_schema.py -v`
Expected: `3 passed`.

- [ ] **Step 9: Commit**

```bash
git add docs/design/review-state.schema.yaml scripts/validate_state.py tests/
git commit -m "feat(m1): finalize review-state schema as JSON Schema + validator and tests"
```

---

### Task 2: Intake skill

**Files:**
- Create: `skills/intake/SKILL.md`

**Interfaces:**
- Consumes: schema + `validate_state.py` (Task 1). GitHub via `gh` CLI.
- Produces: a seeded, schema-valid `review-state.yaml` (all Task 1 field names), the PR worktree at `<state_dir>/worktree`, and `<state_dir>/CLAUDE.md`. Skill invocation contract used by the workflow (Task 7): `tnf-reviewer:intake` with args string `pr_url=<url> state_dir=<abs> posting_mode=<hold|auto> [repos_root=<abs>] [context_root=<abs>]`.

- [ ] **Step 1: Write the skill**

Create `skills/intake/SKILL.md`:

````markdown
---
name: intake
description: "TNF review pipeline stage 1 — resolve a PR, build the context-ful review environment (worktree + domain context index), and seed review-state.yaml. Args: pr_url=<url> state_dir=<abs> posting_mode=hold|auto [repos_root=<abs>] [context_root=<abs>]"
argument-hint: "pr_url=<url> state_dir=<abs-path> posting_mode=hold|auto"
user-invocable: true
---

# tnf-reviewer:intake — stage 1

Builds the environment every later stage runs in, and seeds the state file.
All durable output lands in `<state_dir>/review-state.yaml` — the pipeline bus.
**The review sees the domain, not just the diff**: this stage wires the domain
context in so later stages inherit it.

## Arguments

- `pr_url` (required): `https://github.com/<owner>/<repo>/pull/<number>`.
  Also accept `owner/repo#N` and normalize it.
- `state_dir` (required, absolute): per-PR state directory. Create it if missing.
- `posting_mode` (default `hold`): recorded verbatim in state.
- `repos_root` (default: `/repos` if that directory exists, otherwise the
  workspace `repos/` directory containing the PR's repo clone).
- `context_root` (default: `/context` if it exists, otherwise null): root of
  the vendored TNF domain context (`<context_root>/tnf/*.md`).

## Procedure

1. **Existing-state guard.** If `<state_dir>/review-state.yaml` exists and parses:
   - Fetch PR head with `gh pr view <pr_url> --json headRefOid`. If it equals the
     file's `head_sha`: do NOT reseed (findings and stage statuses are preserved);
     repair the worktree (step 4 only), set `stages.intake` to `done`, refresh
     `updated_at`, validate, and finish.
   - If the head differs: delta re-review is not implemented (M3). Set
     `stages.intake.status: failed` with notes
     `"head moved (<old> -> <new>); delta re-review lands in M3 — use a fresh state_dir"`,
     validate, and report failure.
2. **PR metadata.** `gh pr view <pr_url> --json url,number,baseRefName,headRefOid,author,title,body,files`.
3. **repos_in_scope.** Always include the PR's own repo (short name). Add any of
   `cluster-etcd-operator`, `machine-config-operator`, `resource-agents`,
   `fence-agents`, `api` that the changed file paths, PR title, or PR body
   plausibly implicate (e.g. an OCF agent env var rename implicates
   `resource-agents`). Judgment call — record why in `stages.intake.notes`.
4. **Worktree** (detached, disposable — state records the SHA, not the checkout):
   - `git -C <repos_root>/<repo-short> fetch origin pull/<number>/head` and
     `git -C <repos_root>/<repo-short> fetch origin <base_ref>`.
   - If `<state_dir>/worktree` exists but `git -C <state_dir>/worktree status` fails
     (stale from a dead container), `rm -rf <state_dir>/worktree` and
     `git -C <repos_root>/<repo-short> worktree prune`.
   - `git -C <repos_root>/<repo-short> worktree add --detach <state_dir>/worktree <headRefOid>`.
5. **State-dir index.** Write `<state_dir>/CLAUDE.md` — a lean pointer file for
   ad-hoc resume sessions (do not copy content into it):
   - what this directory is (one review run of `<pr_url>`),
   - paths: `review-state.yaml`, `worktree/`, `findings.md` (exists after compose),
   - the domain context files that exist for each repo in scope: check, in order,
     `<context_root>/tnf/context.md`, `<context_root>/tnf/architecture.md`,
     `<context_root>/tnf/debugging.md`, and each repo's own `CLAUDE.md` /
     `DOMAIN-CONTEXT.md` under `<repos_root>/<repo>/` — list only files that exist,
   - how to resume a held review: "skim `findings.md`, then run
     `tnf-reviewer:post state_dir=<state_dir>`".
6. **Seed state.** Write `<state_dir>/review-state.yaml`:
   - `schema_version: 1`; `pr` block from step 2 (`repo` as `owner/name`);
     `head_sha: <headRefOid>`; `reviewed_sha: null`; `previous_shas: []`;
   - `posting_mode`; `created_at`/`updated_at` = now (UTC ISO-8601);
   - `repos_in_scope`; `tracker: {type: none, id: null, url: null}`;
   - `environment: {repos_root, worktree_path: <state_dir>/worktree, context_root}`;
   - all eight stages `{status: pending, started_at: null, finished_at: null, notes: null}`,
     except `spec-compliance`, `cross-repo`, `arch-fit` seeded
     `{status: skipped, notes: "M2"}`;
   - `findings: []`;
   - `review: {verdict_recommendation: null, body: null, summary_comment_id: null, submitted: false, submitted_at: null}`.
   Then set `stages.intake` to `done` with `started_at`/`finished_at`.
7. **Validate.** `python3 "${CLAUDE_PLUGIN_ROOT}/scripts/validate_state.py" <state_dir>/review-state.yaml`
   must print `OK`. Fix the state file until it does. Report: state file path,
   head SHA, repos in scope, worktree path.
````

- [ ] **Step 2: Pick the replay PR for the shared eval directory**

```bash
gh pr list -R openshift/cluster-etcd-operator --state merged --search "tnf" --limit 10
```

Choose a recently **merged** PR that touches TNF paths (`pkg/tnf/`, fencing, podman-etcd handover); prefer a small-to-medium diff. Record the choice:

```bash
mkdir -p ~/tnf-reviews-eval
echo "eval PR: <url>  (chosen $(date -u +%F))" >> ~/tnf-reviews-eval/eval-notes.md
```

- [ ] **Step 3: Run the skill standalone against the replay PR**

From the worktree root (so the plugin loads), in a fresh headless session:

```bash
cd /Users/pfontani/Workspace/tnf-dev-env/repos/tnf-auto-reviewer/.worktrees/m1-skeleton
claude -p "Use the tnf-reviewer:intake skill with: pr_url=<chosen-pr-url> state_dir=$HOME/tnf-reviews-eval/pr-cluster-etcd-operator-<N> posting_mode=hold repos_root=/Users/pfontani/Workspace/tnf-dev-env/repos" \
  --plugin-dir . --permission-mode acceptEdits \
  --allowedTools "Bash(gh pr view *),Bash(git *),Bash(python3 *),Bash(mkdir *),Bash(rm -rf $HOME/tnf-reviews-eval/*),Bash(date *)"
```

Expected: reports state file path, head SHA, repos in scope, worktree path.

- [ ] **Step 4: Verify the outputs**

```bash
python3 scripts/validate_state.py ~/tnf-reviews-eval/pr-cluster-etcd-operator-<N>/review-state.yaml
git -C ~/tnf-reviews-eval/pr-cluster-etcd-operator-<N>/worktree log -1 --format=%H
test -f ~/tnf-reviews-eval/pr-cluster-etcd-operator-<N>/CLAUDE.md && echo INDEX-OK
```

Expected: `OK`; the worktree HEAD equals `head_sha` in the state file; `INDEX-OK`.

- [ ] **Step 5: Verify the idempotency guard**

Re-run the exact Step 3 command. Expected: reports "state already seeded / head unchanged" behavior — `findings` and stage statuses untouched, no reseed, validator still `OK`.

- [ ] **Step 6: Commit**

```bash
git add skills/intake/SKILL.md
git commit -m "feat(m1): intake stage skill — context-ful env + state seeding"
```

---

### Task 3: Harvest skill (stage key `code-review`) + output fixture

**Files:**
- Create: `skills/harvest/SKILL.md`
- Create: `tests/fixtures/code-review-output.md`

**Interfaces:**
- Consumes: state seeded by `tnf-reviewer:intake` (`environment.worktree_path`, `pr.base_ref`, `head_sha`); the built-in `code-review` skill.
- Produces: `findings[]` entries with `source: code-review`, sequential ids `F-NNN`, `vet.verdict: pending`, `posting.decision: pending`; `stages.code-review: done`. Invocation contract: `tnf-reviewer:harvest` with `state_dir=<abs>`.

- [ ] **Step 1: Write the skill**

Create `skills/harvest/SKILL.md`:

````markdown
---
name: harvest
description: "TNF review pipeline stage 2 (state key: code-review) — run the built-in code-review skill inside the PR worktree and harvest its findings into review-state.yaml. Args: state_dir=<abs>"
argument-hint: "state_dir=<abs-path>"
user-invocable: true
---

# tnf-reviewer:harvest — stage 2 (`stages.code-review`)

Runs the generic code review and translates its output into state. Only this
skill knows the built-in skill's output format — `tests/fixtures/code-review-output.md`
pins that coupling: if the built-in skill's output shape changes, update the
fixture and the mapping below together.

## Procedure

1. Read `<state_dir>/review-state.yaml`. Require `stages.intake.status: done`
   (otherwise set `stages.code-review: failed`, notes "intake not done", and stop).
   Set `stages.code-review` to `running` with `started_at`.
2. From `environment.worktree_path`, invoke the built-in **code-review** skill
   (Skill tool, `skill: code-review`) at effort **medium** over the worktree's
   diff against `origin/<pr.base_ref>`. The worktree is detached at the PR head,
   so the diff `origin/<base_ref>...HEAD` is the PR.
3. **Harvest** every reported finding into `findings[]`:
   - `id`: next sequential `F-NNN` (max existing numeric suffix + 1, zero-padded
     to 3; start at `F-001`).
   - `source: code-review`.
   - `severity` mapping: correctness defects that can crash, lose data, or break
     quorum/fencing/handover → `critical`; other confirmed correctness bugs →
     `major`; efficiency / test-coverage / simplification → `minor`;
     naming/docs/comment issues → `clarity`.
   - `category`: closest of `bug | security | pattern | clarity | minor`.
   - `title`: one line. `detail`: the full finding text including the failure
     scenario, verbatim enough that vet can re-check it without re-running review.
   - `anchor`: `{file: <repo-relative>, line: <int>, side: RIGHT, sha: <head_sha>, diff_position: null}`.
   - `vet`: `{verdict: pending, rationale: null, vetted_at: null, round: 1}`.
   - `lifecycle: active`.
   - `posting`: `{decision: pending, comment_body: null, comment_id: null, posted_at: null}`.
4. Zero findings is a valid outcome (state stays valid with `findings: []`).
5. Set `stages.code-review` to `done`, `finished_at`, notes `"harvested N findings"`.
   Refresh `updated_at`. Validate with
   `python3 "${CLAUDE_PLUGIN_ROOT}/scripts/validate_state.py"` → must print `OK`.
   Report the finding count and severity breakdown.
````

- [ ] **Step 2: Run the skill standalone on the eval directory**

```bash
cd /Users/pfontani/Workspace/tnf-dev-env/repos/tnf-auto-reviewer/.worktrees/m1-skeleton
claude -p "Use the tnf-reviewer:harvest skill with: state_dir=$HOME/tnf-reviews-eval/pr-cluster-etcd-operator-<N>" \
  --plugin-dir . --permission-mode acceptEdits \
  --allowedTools "Bash(git *),Bash(python3 *),Bash(date *)"
```

Expected: reports "harvested N findings" (N ≥ 0; on a real TNF PR usually ≥ 1).

- [ ] **Step 3: Capture the format fixture**

From the session output of Step 2 (or by running the built-in `code-review` skill directly in the eval worktree), save the **raw code-review output** — the findings as the built-in skill reported them — verbatim into `tests/fixtures/code-review-output.md`, with a header comment:

```markdown
<!-- Captured output of the built-in code-review skill (effort: medium), <date>.
     Pins the format the harvest skill maps into review-state.yaml findings.
     If harvest starts mis-mapping, re-capture this fixture and update the
     mapping table in skills/harvest/SKILL.md together. -->
```

- [ ] **Step 4: Verify the harvested state**

```bash
python3 scripts/validate_state.py ~/tnf-reviews-eval/pr-cluster-etcd-operator-<N>/review-state.yaml
python3 - <<'EOF'
import yaml, os
s = yaml.safe_load(open(os.path.expanduser("~/tnf-reviews-eval/pr-cluster-etcd-operator-<N>/review-state.yaml")))
assert s["stages"]["code-review"]["status"] == "done"
assert all(f["source"] == "code-review" and f["vet"]["verdict"] == "pending" for f in s["findings"])
ids = [f["id"] for f in s["findings"]]
assert ids == [f"F-{i+1:03d}" for i in range(len(ids))], ids
print(f"HARVEST-OK: {len(ids)} findings")
EOF
```

Expected: `OK` then `HARVEST-OK: N findings`.

- [ ] **Step 5: Commit**

```bash
git add skills/harvest/SKILL.md tests/fixtures/code-review-output.md
git commit -m "feat(m1): harvest stage skill + code-review output fixture"
```

---

### Task 4: Vet skill (round 1)

**Files:**
- Create: `skills/vet/SKILL.md`

**Interfaces:**
- Consumes: `findings[]` with `vet.verdict: pending` (Task 3); the worktree.
- Produces: every finding vetted `confirmed | dropped` with rationale; `stages.vet: done`. Invocation contract: `tnf-reviewer:vet` with `state_dir=<abs> round=1`.

- [ ] **Step 1: Write the skill**

Create `skills/vet/SKILL.md`:

````markdown
---
name: vet
description: "TNF review pipeline stage 3 — adversarial vet of pending findings against the actual code in the PR worktree; writes confirmed/dropped verdicts with rationale. Args: state_dir=<abs> round=1|2"
argument-hint: "state_dir=<abs-path> round=1"
user-invocable: true
---

# tnf-reviewer:vet — stage 3

A skeptical second pass. The upstream reviewer is enthusiastic; your job is to
kill findings that don't survive contact with the real code — and to keep the
ones that do. Verdicts and rationale are the pipeline's audit trail.

## Procedure

1. Read `<state_dir>/review-state.yaml`. Require `stages.code-review.status: done`
   when `round=1` (fail the stage otherwise). Set `stages.vet` to `running`.
2. For **each** finding with `vet.verdict: pending`:
   - Open `anchor.file` around `anchor.line` in `environment.worktree_path`. Read
     enough surrounding code (callers, guards, error paths) to independently
     re-derive or refute the failure scenario in `detail`. Judge the code **as it
     is at this head**, not the diff hunk in isolation — grep for guards or
     callers elsewhere that neutralize the issue.
   - `confirmed`: the defect is real at this head and the failure scenario holds.
   - `dropped`: demonstrably wrong, already handled elsewhere, describes code
     that doesn't exist at this head, or pure style noise below `clarity` value.
   - **When in doubt, confirm** — the conservative default from
     `agents/reviewer.md` (keep a finding rather than guess it away).
   - Write `vet: {verdict, rationale: <1-3 sentences citing file:line evidence>,
     vetted_at: <now UTC>, round: <round arg>}`.
3. Never delete a finding — dropped ones feed the transparency table in compose.
4. Set `stages.vet` to `done`, notes `"confirmed X, dropped Y"`. Refresh
   `updated_at`. Validate with
   `python3 "${CLAUDE_PLUGIN_ROOT}/scripts/validate_state.py"` → `OK`. Report X/Y.
````

- [ ] **Step 2: Run the skill standalone on the eval directory**

```bash
cd /Users/pfontani/Workspace/tnf-dev-env/repos/tnf-auto-reviewer/.worktrees/m1-skeleton
claude -p "Use the tnf-reviewer:vet skill with: state_dir=$HOME/tnf-reviews-eval/pr-cluster-etcd-operator-<N> round=1" \
  --plugin-dir . --permission-mode acceptEdits \
  --allowedTools "Bash(git *),Bash(python3 *),Bash(date *),Bash(grep *),Bash(rg *)"
```

Expected: reports "confirmed X, dropped Y" with X+Y = the Task 3 finding count.

- [ ] **Step 3: Verify verdicts**

```bash
python3 scripts/validate_state.py ~/tnf-reviews-eval/pr-cluster-etcd-operator-<N>/review-state.yaml
python3 - <<'EOF'
import yaml, os
s = yaml.safe_load(open(os.path.expanduser("~/tnf-reviews-eval/pr-cluster-etcd-operator-<N>/review-state.yaml")))
assert s["stages"]["vet"]["status"] == "done"
for f in s["findings"]:
    assert f["vet"]["verdict"] in ("confirmed", "dropped"), f["id"]
    assert f["vet"]["rationale"], f"{f['id']} missing rationale"
    assert f["vet"]["vetted_at"], f"{f['id']} missing vetted_at"
print("VET-OK")
EOF
```

Expected: `OK` then `VET-OK`. Also **spot-read two rationales** against the worktree code — they must cite real file:line evidence, not restate the finding.

- [ ] **Step 4: Commit**

```bash
git add skills/vet/SKILL.md
git commit -m "feat(m1): vet stage skill — adversarial round-1 verdicts"
```

---

### Task 5: Compose skill

**Files:**
- Create: `skills/compose/SKILL.md`

**Interfaces:**
- Consumes: vetted `findings[]` (Task 4); verdict policy from `agents/reviewer.md`.
- Produces: `review.body` + per-finding `posting.decision`/`posting.comment_body` (each inline body ends with marker `<!-- tnf-reviewer:F-NNN -->`), `review.verdict_recommendation`, `<state_dir>/findings.md`; `stages.compose: done`. Invocation contract: `tnf-reviewer:compose` with `state_dir=<abs>`.

- [ ] **Step 1: Write the skill**

Create `skills/compose/SKILL.md`:

````markdown
---
name: compose
description: "TNF review pipeline stage 4 — compose the complete GitHub review in state (inline comment bodies, summary body, verdict recommendation, dropped-findings table) and render findings.md. Args: state_dir=<abs>"
argument-hint: "state_dir=<abs-path>"
user-invocable: true
---

# tnf-reviewer:compose — stage 4

Builds the **exact** review that will be posted. After this stage, posting is
mechanical: what sits in state is what lands on GitHub, verbatim.

## Procedure

1. Read `<state_dir>/review-state.yaml`. Require `stages.vet.status: done`.
   Set `stages.compose` to `running`.
2. **Posting decision** per finding:
   - `confirmed` + severity `critical` or `major` → `posting.decision: inline`.
   - `confirmed` + severity `minor` or `clarity` → `posting.decision: summary`.
   - `dropped` → `posting.decision: suppressed`.
3. **Inline comment bodies.** For each `inline` finding write
   `posting.comment_body`: the finding in reviewer voice — what's wrong, the
   concrete failure scenario, a suggested direction (no prescriptive patches) —
   ending with the marker on its own line: `<!-- tnf-reviewer:F-NNN -->`
   (invisible on GitHub; it is the idempotency/reconciliation key).
4. **Verdict** (mechanical policy, `agents/reviewer.md`): any `confirmed`
   `critical` or `major` finding → `REQUEST_CHANGES`; otherwise → `COMMENT`.
   **Never APPROVE** — a clean read is still `COMMENT`.
5. **`review.body`** (the top-level review text), in order:
   - one-paragraph summary of the PR and the review outcome;
   - `### Minor notes` — a bullet per `summary` finding (`file:line` + one line);
   - `**Recommendation:** REQUEST_CHANGES|COMMENT` + one-line justification;
   - `### Dropped as noise` — table `| ID | Title | Why dropped |` from the
     `suppressed` findings (omit the section if none);
   - footer: `<!-- tnf-reviewer:summary -->` then
     `*Generated by [tnf-auto-reviewer](https://github.com/pablofontanilla/tnf-auto-reviewer) — verdicts and rationale in the run's review-state.yaml.*`
6. **Render `<state_dir>/findings.md`** for the human skim: PR url/head, verdict
   recommendation, each inline comment with its `file:line` anchor, the minor
   notes, the dropped table. It is a rendering of state — never edit it by hand.
7. Set `stages.compose` to `done`. Refresh `updated_at`. Validate with
   `python3 "${CLAUDE_PLUGIN_ROOT}/scripts/validate_state.py"` → `OK`.
   Report: verdict, inline/summary/suppressed counts, findings.md path.
````

- [ ] **Step 2: Run the skill standalone on the eval directory**

```bash
cd /Users/pfontani/Workspace/tnf-dev-env/repos/tnf-auto-reviewer/.worktrees/m1-skeleton
claude -p "Use the tnf-reviewer:compose skill with: state_dir=$HOME/tnf-reviews-eval/pr-cluster-etcd-operator-<N>" \
  --plugin-dir . --permission-mode acceptEdits \
  --allowedTools "Bash(python3 *),Bash(date *)"
```

Expected: reports verdict + counts + findings.md path.

- [ ] **Step 3: Verify the composition**

```bash
python3 scripts/validate_state.py ~/tnf-reviews-eval/pr-cluster-etcd-operator-<N>/review-state.yaml
python3 - <<'EOF'
import yaml, os
d = os.path.expanduser("~/tnf-reviews-eval/pr-cluster-etcd-operator-<N>")
s = yaml.safe_load(open(f"{d}/review-state.yaml"))
assert s["stages"]["compose"]["status"] == "done"
assert s["review"]["verdict_recommendation"] in ("REQUEST_CHANGES", "COMMENT")
assert s["review"]["body"] and "<!-- tnf-reviewer:summary -->" in s["review"]["body"]
for f in s["findings"]:
    assert f["posting"]["decision"] in ("inline", "summary", "suppressed")
    if f["posting"]["decision"] == "inline":
        assert f"<!-- tnf-reviewer:{f['id']} -->" in f["posting"]["comment_body"]
assert os.path.exists(f"{d}/findings.md")
print("COMPOSE-OK")
EOF
```

Expected: `OK` then `COMPOSE-OK`. Also read `findings.md` yourself — it must be skimmable in under a minute.

- [ ] **Step 4: Commit**

```bash
git add skills/compose/SKILL.md
git commit -m "feat(m1): compose stage skill — verbatim review body + findings.md"
```

---

### Task 6: Post skill (verbatim, never approve)

**Files:**
- Create: `skills/post/SKILL.md`

**Interfaces:**
- Consumes: composed `review` + `findings[].posting` (Task 5).
- Produces: a submitted GitHub review (or, with `dry_run=true`, `<state_dir>/review.json` only); `posting.comment_id` per inline finding; `review.submitted: true`; `stages.post: done`. Invocation contract: `tnf-reviewer:post` with `state_dir=<abs> [dry_run=true]` — also the **hold-resume entrypoint** a human runs ad hoc.

- [ ] **Step 1: Write the skill**

Create `skills/post/SKILL.md`:

````markdown
---
name: post
description: "TNF review pipeline stage 5 — submit the composed review to GitHub verbatim from review-state.yaml (COMMENT or REQUEST_CHANGES, never APPROVE); records comment IDs. Also the resume entrypoint for held reviews. Args: state_dir=<abs> [dry_run=true]"
argument-hint: "state_dir=<abs-path> [dry_run=true]"
user-invocable: true
---

# tnf-reviewer:post — stage 5

Deliberately trivial: post **exactly** what compose put in state. No rewriting,
no re-judging. If the review needs changing, fix the findings and re-run
compose — never edit here.

## Procedure

1. Read `<state_dir>/review-state.yaml`. Require `stages.compose.status: done`.
   If `review.submitted` is `true`: report "already submitted" and stop —
   never double-post.
2. **Reconcile before ever posting** (crash-safety): list existing reviews on
   the PR (`gh api repos/<owner>/<name>/pulls/<number>/reviews --paginate`) and
   their comments; if any body contains `<!-- tnf-reviewer:summary -->`, a
   previous run already posted — record its IDs into state (step 5), set
   `review.submitted: true`, and stop.
3. **Build `<state_dir>/review.json`**:
   ```json
   {
     "commit_id": "<head_sha>",
     "event": "<review.verdict_recommendation>",
     "body": "<review.body>",
     "comments": [
       {"path": "<anchor.file>", "line": <anchor.line>, "side": "RIGHT",
        "body": "<posting.comment_body>"}
     ]
   }
   ```
   One entry per finding with `posting.decision: inline` and
   `posting.comment_id: null`. Build it with python3 (yaml in, json out) so the
   bodies survive quoting exactly.
   **Guard:** if `event` is not `COMMENT` or `REQUEST_CHANGES` (e.g. `APPROVE`
   or null), set `stages.post: failed` with notes and stop. The agent never
   approves.
4. If `dry_run=true`: stop here. Leave `stages.post` untouched, report the
   `review.json` path and its comment count. (This is the task-level test hook.)
5. **Post:** `gh api repos/<owner>/<name>/pulls/<number>/reviews --input <state_dir>/review.json`.
   From the response, record the review `id` into `review.summary_comment_id`.
   Then `gh api repos/<owner>/<name>/pulls/<number>/reviews/<id>/comments --paginate`
   and match each comment to its finding by the `<!-- tnf-reviewer:F-NNN -->`
   marker in the body → write `posting.comment_id` (as string) and `posted_at`.
   An inline comment GitHub rejected (e.g. line not in diff): leave its
   `comment_id` null and note the loss in `stages.post.notes` — do not fail the
   whole stage for it.
6. Set `review.submitted: true`, `submitted_at`, `stages.post: done`. Refresh
   `updated_at`. Validate with
   `python3 "${CLAUDE_PLUGIN_ROOT}/scripts/validate_state.py"` → `OK`.
   Report: review URL, event, posted/lost comment counts.
````

- [ ] **Step 2: Dry-run against the eval directory**

```bash
cd /Users/pfontani/Workspace/tnf-dev-env/repos/tnf-auto-reviewer/.worktrees/m1-skeleton
claude -p "Use the tnf-reviewer:post skill with: state_dir=$HOME/tnf-reviews-eval/pr-cluster-etcd-operator-<N> dry_run=true" \
  --plugin-dir . --permission-mode acceptEdits \
  --allowedTools "Bash(gh api *),Bash(python3 *),Bash(date *)"
```

Expected: reports the `review.json` path and comment count; **no** GitHub write happens (the PR is merged — we must not post to it; `dry_run` is mandatory here).

- [ ] **Step 3: Verify review.json**

```bash
python3 - <<'EOF'
import json, yaml, os
d = os.path.expanduser("~/tnf-reviews-eval/pr-cluster-etcd-operator-<N>")
r = json.load(open(f"{d}/review.json"))
s = yaml.safe_load(open(f"{d}/review-state.yaml"))
assert r["event"] in ("COMMENT", "REQUEST_CHANGES")
assert r["body"] == s["review"]["body"]                      # verbatim
inline = [f for f in s["findings"] if f["posting"]["decision"] == "inline"]
assert len(r["comments"]) == len(inline)
for c, f in zip(sorted(r["comments"], key=lambda c: c["body"]),
                sorted(inline, key=lambda f: f["posting"]["comment_body"])):
    assert c["body"] == f["posting"]["comment_body"]          # verbatim
print("POST-DRYRUN-OK")
EOF
```

Expected: `POST-DRYRUN-OK`. (Live posting is exercised in Task 9 against a scratch PR.)

- [ ] **Step 4: Commit**

```bash
git add skills/post/SKILL.md
git commit -m "feat(m1): post stage skill — verbatim submit, marker reconciliation, never approve"
```

---

### Task 7: Dynamic workflow orchestrator

**Files:**
- Create: `.claude/workflows/review.js`

**Interfaces:**
- Consumes: all five stage skills (exact invocation strings from Tasks 2–6); `args` global `{pr_url, state_dir, posting_mode, repos_root?, context_root?}`.
- Produces: the `/tnf-review` workflow command — sequential stages, skip-when-done, hold branch, run-summary string. Used verbatim by Task 8's entrypoint and Task 9's runs.

- [ ] **Step 1: Write the workflow script**

Create `.claude/workflows/review.js`:

```javascript
export const meta = {
  name: 'tnf-review',
  description:
    'TNF PR-review pipeline: intake → code-review harvest → vet → compose → post (auto) or hold. ' +
    'Idempotent over review-state.yaml: stages already done are skipped.',
}

// Invoked as: /tnf-review pr_url=<url> state_dir=<abs> posting_mode=hold|auto
//             [repos_root=<abs>] [context_root=<abs>]
// Workflow scripts have no filesystem access — every disk/network touch below
// happens inside an agent() call; this script only sequences and branches.
const prUrl = args?.pr_url
const stateDir = args?.state_dir
const postingMode = args?.posting_mode ?? 'hold'
const reposRoot = args?.repos_root ?? ''
const contextRoot = args?.context_root ?? ''
if (!prUrl || !stateDir) throw new Error('required args: pr_url, state_dir')
if (postingMode !== 'hold' && postingMode !== 'auto')
  throw new Error(`posting_mode must be hold or auto, got: ${postingMode}`)
const stateFile = `${stateDir}/review-state.yaml`

const M1_STAGES = ['intake', 'code-review', 'vet', 'compose', 'post']

// -- read current stage statuses (a cheap agent does the file I/O) ----------
const statusSchema = {
  type: 'object',
  required: ['exists', 'stages'],
  properties: {
    exists: { type: 'boolean' },
    stages: {
      type: 'object',
      properties: Object.fromEntries(M1_STAGES.map((k) => [k, { type: 'string' }])),
    },
  },
}
const initial = await agent(
  `If the file ${stateFile} does not exist, return exists=false and an empty stages object. ` +
    `Otherwise read it and return exists=true plus, for each of: ${M1_STAGES.join(', ')}, ` +
    `the value of stages.<name>.status. Strictly read-only — change nothing.`,
  { schema: statusSchema, label: 'read-state' },
)

// -- stage runner ------------------------------------------------------------
const stageSchema = {
  type: 'object',
  required: ['status'],
  properties: {
    status: { type: 'string' }, // done | failed
    notes: { type: 'string' },
  },
}
const summary = []
async function runStage(stageKey, prompt) {
  if (initial.exists && initial.stages[stageKey] === 'done') {
    summary.push(`${stageKey}: skipped (already done)`)
    return
  }
  const res = await agent(prompt, { schema: stageSchema, label: stageKey })
  summary.push(`${stageKey}: ${res.status}${res.notes ? ` — ${res.notes}` : ''}`)
  if (res.status !== 'done') {
    // Never post a partial review: stop the run at the first failed stage.
    throw new Error(`stage ${stageKey} did not complete (status=${res.status}).\n${summary.join('\n')}`)
  }
}

const common =
  `The state file is ${stateFile}. Follow the skill exactly; when finished the state file must ` +
  `validate (the skill runs scripts/validate_state.py). Return status=done with a one-line notes ` +
  `summary on success. If the stage cannot complete, make sure its stages entry says failed with ` +
  `notes in the state file, then return status=failed with notes.`

await runStage(
  'intake',
  `Use the tnf-reviewer:intake skill with: pr_url=${prUrl} state_dir=${stateDir} ` +
    `posting_mode=${postingMode}` +
    (reposRoot ? ` repos_root=${reposRoot}` : '') +
    (contextRoot ? ` context_root=${contextRoot}` : '') +
    `. ${common}`,
)
await runStage('code-review', `Use the tnf-reviewer:harvest skill with: state_dir=${stateDir}. ${common}`)
await runStage('vet', `Use the tnf-reviewer:vet skill with: state_dir=${stateDir} round=1. ${common}`)
await runStage('compose', `Use the tnf-reviewer:compose skill with: state_dir=${stateDir}. ${common}`)

if (postingMode === 'auto') {
  await runStage('post', `Use the tnf-reviewer:post skill with: state_dir=${stateDir}. ${common}`)
} else if (initial.exists && initial.stages.post === 'done') {
  summary.push('post: skipped (already done)')
} else {
  await agent(
    `Edit ${stateFile}: set stages.post.status to "held" (leave its other fields), refresh ` +
      `updated_at (UTC ISO-8601), then run python3 scripts/validate_state.py ${stateFile} and ` +
      `confirm it prints OK. Return status=done.`,
    { schema: stageSchema, label: 'hold' },
  )
  summary.push(
    `post: held — composed review is in ${stateDir}/findings.md; ` +
      `resume with: tnf-reviewer:post state_dir=${stateDir}`,
  )
}

return `tnf-review run (posting_mode=${postingMode})\n${summary.join('\n')}`
```

- [ ] **Step 2: Run the workflow end-to-end on a fresh state dir**

Use a **second** state directory (same replay PR) so the run exercises every stage from scratch:

```bash
cd /Users/pfontani/Workspace/tnf-dev-env/repos/tnf-auto-reviewer/.worktrees/m1-skeleton
claude -p "/tnf-review pr_url=<chosen-pr-url> state_dir=$HOME/tnf-reviews-eval/wf-run-1 posting_mode=hold repos_root=/Users/pfontani/Workspace/tnf-dev-env/repos" \
  --plugin-dir . --permission-mode acceptEdits \
  --allowedTools "Bash(gh pr view *),Bash(gh api *),Bash(git *),Bash(python3 *),Bash(mkdir *),Bash(rm -rf $HOME/tnf-reviews-eval/*),Bash(date *),Bash(grep *),Bash(rg *)"
```

Expected: final output is the run summary — `intake: done … compose: done`, and `post: held — …resume with: tnf-reviewer:post…`.

- [ ] **Step 3: Verify held state**

```bash
python3 scripts/validate_state.py ~/tnf-reviews-eval/wf-run-1/review-state.yaml
python3 -c "
import yaml, os
s = yaml.safe_load(open(os.path.expanduser('~/tnf-reviews-eval/wf-run-1/review-state.yaml')))
assert s['stages']['post']['status'] == 'held', s['stages']['post']
assert s['stages']['compose']['status'] == 'done'
assert s['posting_mode'] == 'hold'
print('HELD-OK')"
```

Expected: `OK` then `HELD-OK`.

- [ ] **Step 4: Verify idempotent restart**

Re-run the exact Step 2 command. Expected: the summary shows `intake: skipped (already done)` … `compose: skipped (already done)` — the second run must be dramatically faster and add no findings (`python3 -c` compare finding count before/after).

- [ ] **Step 5: Commit**

```bash
git add .claude/workflows/review.js
git commit -m "feat(m1): tnf-review dynamic workflow — sequenced stages, idempotent over state"
```

---

### Task 8: Run-skill rewrite + README status

**Files:**
- Rewrite: `skills/run/SKILL.md`
- Modify: `README.md` (status paragraph + layout block)

**Interfaces:**
- Consumes: the `/tnf-review` workflow (Task 7); stage skills (Tasks 2–6).
- Produces: `tnf-reviewer:run` — the thin interactive entrypoint documented in the design ("`/tnf-reviewer:run` remains the thin entrypoint that launches the workflow").

- [ ] **Step 1: Rewrite the run skill**

Replace the entire contents of `skills/run/SKILL.md`:

````markdown
---
name: run
description: "Run the TNF PR-review pipeline on a PR: launches the tnf-review dynamic workflow over review-state.yaml. Args: <PR URL | owner/repo#N> [--posting-mode=hold|auto]"
argument-hint: "<PR URL | owner/repo#number> [--posting-mode=hold|auto]"
user-invocable: true
---

# /tnf-reviewer:run — pipeline entrypoint

Thin wrapper: resolve arguments, compute the state directory, launch the
`tnf-review` dynamic workflow. All pipeline logic lives in the workflow
(`.claude/workflows/review.js`) and the stage skills; all judgment policy in
`agents/reviewer.md`.

## Procedure

1. **Parse args.** PR reference (required): a full URL, or `owner/repo#N` →
   normalize to `https://github.com/<owner>/<repo>/pull/<N>`.
   `--posting-mode` (optional, default `hold`).
2. **State root:** `$TNF_REVIEW_STATE_ROOT` if set, else `~/tnf-reviews`
   (create it). **State dir:** `<root>/pr-<repo-short-name>-<number>`.
3. **Launch the workflow:** run `/tnf-review` with
   `pr_url=<url> state_dir=<abs state dir> posting_mode=<mode>`
   (add `repos_root=`/`context_root=` if the caller provided them). The
   workflow requires this session to have the repo's `.claude/workflows/`
   loaded — run from the plugin repo, or in the container.
   *Fallback* (workflows unavailable in this session): run the stage skills
   yourself, in order, with the same state_dir — `tnf-reviewer:intake` (with
   pr_url + posting_mode), `tnf-reviewer:harvest`, `tnf-reviewer:vet` (round=1),
   `tnf-reviewer:compose`, then `tnf-reviewer:post` only when posting_mode=auto
   (in hold mode set `stages.post.status: held` instead). Stop at the first
   stage that reports failure.
4. **Report** the run summary. If the run ended `held`, tell the user:
   skim `<state_dir>/findings.md`, then post with
   `tnf-reviewer:post state_dir=<state_dir>`.
````

- [ ] **Step 2: Update README status**

In `README.md`, replace the `> **Status:** scaffolding…` blockquote with:

```markdown
> **Status:** M1 walking skeleton implemented — unattended pipeline
> (intake → code-review harvest → vet → compose) over `review-state.yaml`,
> `hold`/`auto` posting via the `tnf-review` dynamic workflow, podman runtime
> image. M2 (Jira signal + context passes) and M3 (delta re-review, auto-mode
> hardening) follow the PoC spec: `docs/design/2026-07-12-poc-design.md`.
```

and update the `## Layout` block to match the File Structure section of this plan (add `.claude/workflows/review.js`, `Containerfile`, `entrypoint.sh`, `context/tnf/`, `scripts/validate_state.py`, `skills/{intake,harvest,vet,compose,post}`, `tests/`).

- [ ] **Step 3: Verify the fallback path documentation is honest**

Run: `grep -n "tnf-reviewer:" skills/run/SKILL.md` and confirm every referenced skill name exists under `skills/` (`intake`, `harvest`, `vet`, `compose`, `post`).
Expected: all five resolve; no stale names.

- [ ] **Step 4: Commit**

```bash
git add skills/run/SKILL.md README.md
git commit -m "feat(m1): run entrypoint skill launches tnf-review workflow; README status"
```

---

### Task 9: Container image, entrypoint, vendored context, and M1 acceptance

**Files:**
- Create: `Containerfile`
- Create: `entrypoint.sh`
- Create: `context/tnf/context.md`, `context/tnf/architecture.md`, `context/tnf/debugging.md` (vendored copies)
- Modify: `.gitignore` (if it excludes `context/` or `*.sh`, un-exclude)

**Interfaces:**
- Consumes: everything (plugin, workflow, skills, validator).
- Produces: image `tnf-reviewer:m1`; run contract `podman run --rm -v <state-root>:/state -e PR_URL -e POSTING_MODE -e GITHUB_TOKEN -e ANTHROPIC_API_KEY tnf-reviewer:m1`.

- [ ] **Step 1: Vendor the TNF context** (PoC decision: hardcoding is acceptable; the image must be self-contained)

```bash
mkdir -p context/tnf
cp /Users/pfontani/Workspace/tnf-dev-env/domains/tnf/context.md context/tnf/context.md
cp /Users/pfontani/Workspace/tnf-dev-env/domains/tnf/docs/architecture.md context/tnf/architecture.md
cp /Users/pfontani/Workspace/tnf-dev-env/domains/tnf/docs/debugging.md context/tnf/debugging.md
```

Add a provenance header to each copied file (first line): `<!-- Vendored from workspace domains/tnf on 2026-07-12 (PoC hardcoding; refresh manually). -->`

- [ ] **Step 2: Write the entrypoint**

Create `entrypoint.sh`:

```bash
#!/usr/bin/env bash
# One container run = one pipeline pass over one PR.
set -euo pipefail

if [[ "${SMOKE:-0}" == "1" ]]; then
  claude --version && gh --version | head -1 && git --version && python3 --version
  exit 0
fi

: "${PR_URL:?PR_URL is required (https://github.com/<owner>/<repo>/pull/<n>)}"
: "${GITHUB_TOKEN:?GITHUB_TOKEN is required}"
: "${ANTHROPIC_API_KEY:?ANTHROPIC_API_KEY is required}"
POSTING_MODE="${POSTING_MODE:-hold}"
STATE_ROOT="${STATE_ROOT:-/state}"

# The baked clones are a cache, not a snapshot: refresh before every run.
for r in /repos/*/; do
  git -C "$r" fetch --quiet origin || echo "warn: fetch failed for $r (continuing on baked state)" >&2
done

repo_short="$(echo "$PR_URL" | sed -E 's#https://github.com/[^/]+/([^/]+)/pull/[0-9]+.*#\1#')"
pr_num="$(echo "$PR_URL" | sed -E 's#.*/pull/([0-9]+).*#\1#')"
state_dir="$STATE_ROOT/pr-$repo_short-$pr_num"
mkdir -p "$state_dir"

cd /opt/tnf-reviewer
exec claude -p "/tnf-review pr_url=$PR_URL state_dir=$state_dir posting_mode=$POSTING_MODE repos_root=/repos context_root=/context" \
  --plugin-dir /opt/tnf-reviewer \
  --permission-mode bypassPermissions \
  --output-format json
```

`--permission-mode bypassPermissions` is acceptable here and only here: the container is single-purpose and isolated, and credentials scope what it can reach.

- [ ] **Step 3: Write the Containerfile**

Create `Containerfile`:

```dockerfile
FROM registry.fedoraproject.org/fedora:42

# Base tooling. gh comes from Fedora's repos; yq is a static binary.
RUN dnf install -y git gh jq ripgrep python3 python3-pyyaml python3-jsonschema && dnf clean all
RUN curl -fsSL -o /usr/local/bin/yq \
      https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64 && \
    chmod +x /usr/local/bin/yq

# Claude Code CLI (dynamic workflows require >= 2.1.154; installer ships latest).
RUN curl -fsSL https://claude.ai/install.sh | bash && \
    ln -s /root/.local/bin/claude /usr/local/bin/claude

# Baked layer: blobless TNF repo clones (full history for later arch-fit; the
# entrypoint fetches on every run, so the image is a cache, not a snapshot).
RUN mkdir -p /repos && cd /repos && \
    git clone --filter=blob:none https://github.com/openshift/cluster-etcd-operator.git && \
    git clone --filter=blob:none https://github.com/openshift/machine-config-operator.git && \
    git clone --filter=blob:none https://github.com/openshift/api.git && \
    git clone --filter=blob:none https://github.com/ClusterLabs/resource-agents.git && \
    git clone --filter=blob:none https://github.com/ClusterLabs/fence-agents.git

# Vendored TNF domain context + this plugin. Credentials are NEVER baked:
# GITHUB_TOKEN / ANTHROPIC_API_KEY / model config are injected at run time.
COPY context/ /context/
COPY . /opt/tnf-reviewer
RUN chmod +x /opt/tnf-reviewer/entrypoint.sh

WORKDIR /opt/tnf-reviewer
ENTRYPOINT ["/opt/tnf-reviewer/entrypoint.sh"]
```

- [ ] **Step 4: Build and smoke-test**

```bash
podman build -t tnf-reviewer:m1 .
podman run --rm -e SMOKE=1 tnf-reviewer:m1
```

Expected: build succeeds (repo clones take a few minutes); smoke prints four version lines and exits 0. If `dnf install gh` fails on the chosen Fedora tag, install gh from the cli.github.com RPM repo instead and note it in the Containerfile.

- [ ] **Step 5: Containerized replay run (hold)**

```bash
mkdir -p ~/tnf-reviews
podman run --rm -v ~/tnf-reviews:/state:Z \
  -e PR_URL=<chosen-replay-pr-url> -e POSTING_MODE=hold \
  -e GITHUB_TOKEN="$(gh auth token)" -e ANTHROPIC_API_KEY \
  tnf-reviewer:m1
```

Expected: JSON output whose `result` contains the run summary ending in `post: held — …`; then on the host:

```bash
python3 scripts/validate_state.py ~/tnf-reviews/pr-<repo-short>-<n>/review-state.yaml
cat ~/tnf-reviews/pr-<repo-short>-<n>/findings.md
```

Expected: `OK`, and a skimmable findings.md.

- [ ] **Step 6: Acceptance — full loop with a live post on a scratch PR**

Never post to the merged replay PR. Use a scratch PR on the user's own repo:

```bash
# scratch branch + PR on pablofontanilla/tnf-auto-reviewer itself
git -C /Users/pfontani/Workspace/tnf-dev-env/repos/tnf-auto-reviewer worktree add .worktrees/scratch-post-test -b scratch-post-test main
cd /Users/pfontani/Workspace/tnf-dev-env/repos/tnf-auto-reviewer/.worktrees/scratch-post-test
echo "scratch line for M1 post acceptance test" >> README.md
git add README.md && git commit -m "test: scratch change for M1 post acceptance"
git push -u origin scratch-post-test
gh pr create --title "test: M1 post acceptance scratch PR" --body "Scratch PR for tnf-reviewer M1 acceptance. Will be closed unmerged." 
```

Container run in hold mode against that PR (same command as Step 5 with the scratch `PR_URL`), then the **ad-hoc resume** — the M1 acceptance moment:

```bash
cd /Users/pfontani/Workspace/tnf-dev-env/repos/tnf-auto-reviewer/.worktrees/m1-skeleton
# 1. Human skim:
cat ~/tnf-reviews/pr-tnf-auto-reviewer-<N>/findings.md
# 2. Post verbatim:
claude -p "Use the tnf-reviewer:post skill with: state_dir=$HOME/tnf-reviews/pr-tnf-auto-reviewer-<N>" \
  --plugin-dir . --permission-mode acceptEdits \
  --allowedTools "Bash(gh api *),Bash(python3 *),Bash(date *)"
# 3. Verify on GitHub:
gh pr view <scratch-pr-url> --json reviews --jq '.reviews[-1].state'   # COMMENT or CHANGES_REQUESTED, NEVER APPROVED
# 4. Verify idempotency: re-run the post command above.
#    Expected: "already submitted", no second review posted.
# 5. Verify state recorded IDs:
python3 -c "
import yaml, os
s = yaml.safe_load(open(os.path.expanduser('~/tnf-reviews/pr-tnf-auto-reviewer-<N>/review-state.yaml')))
assert s['review']['submitted'] is True
assert s['stages']['post']['status'] == 'done'
print('ACCEPTANCE-OK')"
```

Then clean up: `gh pr close <scratch-pr-url> --delete-branch` and `git -C /Users/pfontani/Workspace/tnf-dev-env/repos/tnf-auto-reviewer worktree remove .worktrees/scratch-post-test`.

- [ ] **Step 7: Full test suite green**

Run: `python3 -m pytest tests/ -v`
Expected: all pass.

- [ ] **Step 8: Commit**

```bash
git add Containerfile entrypoint.sh context/ .gitignore
git commit -m "feat(m1): runtime image — baked repos + vendored context, headless entrypoint"
```

- [ ] **Step 9: Wrap up the branch**

Do **not** push without the user: summarize the M1 result (acceptance evidence from Step 6), then follow superpowers:finishing-a-development-branch to decide push/PR with the user. Also update the project workspace docs (`projects/agent-pr-review-pipeline/CLAUDE.md`): check off "Implement M1" and "Implementation started".

---

## Self-Review (completed)

1. **Spec coverage** — M1 deliverables from the PoC spec, mapped: schema finalized (Task 1); intake context-ful env, no Jira (Task 2); `/code-review` + harvest with format fixture (Task 3); vet round 1 (Task 4); compose (Task 5); post-verbatim (Task 6); workflow script (Task 7); `hold` default + `posting_mode` as workflow input recorded in state (Tasks 1, 7, 8); Containerfile with baked/injected split + entrypoint fetch (Task 9); testing story — schema/unit (Task 1), stage-standalone against a replayed merged PR (Tasks 2–6), end-to-end + acceptance incl. no-duplicate posting (Tasks 7, 9); error handling — failed stage stops run, never partial post (workflow `runStage`, post reconciliation). Deliberately out (per spec): Jira/intent.md, context passes, vet round 2 (M2); delta re-review, bake validation, auto-mode hardening (M3). The `auto` branch of the workflow exists and is exercised only via `dry_run`-guarded post logic — earning `auto` on live PRs is M3.
2. **Placeholder scan** — every SKILL.md, script, schema, fixture, Containerfile is written in full above; the only execution-time blanks are the replay-PR choice (`<chosen-pr-url>`, `<N>` — deliberately chosen at Task 2 Step 2 and reused verbatim) and captured fixture content (Task 3 Step 3).
3. **Type consistency** — stage keys (`intake, code-review, vet, spec-compliance, cross-repo, arch-fit, compose, post`), status enum (incl. `held`), finding fields, `environment.{repos_root,worktree_path,context_root}`, marker format `<!-- tnf-reviewer:F-NNN -->`, skill names (`tnf-reviewer:{intake,harvest,vet,compose,post,run}`), workflow name `/tnf-review`, and arg strings are identical across schema, skills, workflow, and entrypoint.
