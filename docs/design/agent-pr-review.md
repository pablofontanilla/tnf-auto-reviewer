# Agent-Based PR Review Pipeline — Design

**Status:** Draft
**Scope:** TNF preset first; generalization via preset config later
**Related:** `docs/design/review-state.schema.yaml` (state artifact schema),
`.claude/skills/vet-review/SKILL.md` (pipeline mode)

## Motivation

PR reviews are ~30% of TNF project work. The manual workflow today:

1. Create a project workspace for the PR (`/project:new`, analysis type)
2. Run a code-review plugin that fans out specialized review agents
3. Run `/vet-review` to filter noise, approving/dismissing each finding
   interactively (~99% of post-vet findings get approved)
4. Optionally run an adversarial pass over the PR's architecture fit
5. Manually post inline comments and a top-level review

This design turns that workflow into an agent pipeline that, in its
final form, runs **completely unattended** and posts its review to the
PR itself (CodeRabbit-style) — no human in the loop, though the agent
never approves. A **hold mode** exists for the evaluation and
development phase: the pipeline runs all stages but stops before
posting, leaving the composed review for a human to skim and submit.
Hold is a trust-building mechanism, not the end state.

## Design principles

1. **Disk is the message bus.** Stages communicate exclusively through
   artifacts in the project workspace — never through shared
   conversation context. Every stage can run in a fresh subagent.
2. **The project workspace is the pipeline's memory.** Resumability,
   auditability, and delta re-review all fall out of the existing
   project conventions (lean index + detail files + frontmatter).
3. **Exploit the multi-repo workspace.** The framework's differentiator
   over a standalone review plugin is that all TNF repos are cloned
   locally — cross-repo contract checking is the unique value-add.
4. **The agent never approves.** On OpenShift repos, Prow gives review
   states and `/lgtm` real gating semantics. Approval stays human.
5. **Every interactive skill needs an autonomous mode** where prompts
   become defaults and decisions become logged rationale.

## Architecture layering

The pipeline is split into three layers. The placement test: *would a
human using this interactively still need it?* If yes → preset. If it
only matters when nobody is watching → agent definition.

| Layer | Artifact | Contains | Knows about |
|-------|----------|----------|-------------|
| 1. Domain preset | `presets/tnf/` | Repos, context files, docs | What the world is |
| 2. Workflow preset | `presets/tnf-reviewer/` (extends `tnf`) | PR-review skills, `review-state.yaml` schema, review parameters | How to review |
| 3. Agent definition | `presets/tnf-reviewer/agents/reviewer.md` | Autonomy policy | When to act without asking |

**Layer 2 — the `tnf-reviewer` preset** is a complete, human-usable
review environment: a person drives the skills one by one with all
interactive gates on (this is the workflow as practiced manually
today, made shareable). Domain parameters live in its `preset.yaml`,
not in the skills:

```yaml
extends: tnf
review:
  contract_repos:            # cross-repo pass greps these for consumers
    - machine-config-operator
    - resource-agents
    - fence-agents
  lifecycle_phases:          # spec-compliance must cover each phase
    - install
    - bootstrap
    - handover
    - steady-state
  tracker: { type: jira, base_url: "https://issues.redhat.com" }
```

Skills read these parameters, so an `lvms-reviewer` preset is the same
workflow skills with different values — no skill changes.

**Layer 3 — the agent definition** is pure policy layered on top: the
entry trigger, stage ordering as pre/postconditions over
`review-state.yaml`, which interactive gates collapse to defaults
(vet → pipeline mode), the verdict policy, the never-approve rule,
escalation criteria (when to stop and ask a human), termination and
idempotency rules. It contains no procedure (skills own that) and no
domain knowledge (the preset owns that). The verdict policy in the
Compose stage below belongs to this layer — a human using the preset
interactively makes their own verdict call.

This layering is the generalization mechanism: **preset = capability,
agent definition = autonomy**. A `tnf-bug-triage` agent is a different
layer-3 file over the same layer-1 preset.

**Mechanism gaps this requires** (feed into the external-presets and
domain-decoupling issues):

1. Presets declare their skills via a **manifest** in `preset.yaml`
   rather than shipping skill files (see Distribution below).
2. The preset schema needs `extends:` (composition) so `tnf-reviewer`
   does not fork `tnf`'s repos and context files. Decide before
   external preset packs exist — retrofitting inheritance later is
   painful.

## Distribution: skill manifest and pre-baked runtime image

### Skill manifest

The workflow preset declares its skills instead of bundling files —
entries can be local (in the preset), a git URL, or a Claude Code
plugin/marketplace reference:

```yaml
skills:
  - name: pr-review-pipeline
    source: local                    # presets/tnf-reviewer/skills/
  - name: code-review
    source: plugin
    ref: anthropics/code-review
    version: "1.4.2"                 # PINNED — the harvest step is
                                     # coupled to this plugin's output
                                     # format; unpinned = silent breakage
```

`/dev-env-setup` resolves the manifest at init time. Local skills can
later move to their own repo without consumers noticing — only the
manifest entry changes.

### Pre-baked runtime image (bake vs. fry)

To make agent startup fast, layers 1+2 are baked into a container
image; only fast-changing specifics are fetched at run time:

| When | What |
|------|------|
| **Baked** (image build) | Cloned repos, distributed context files, skills installed per manifest, bake validation |
| **Fetched** (run start) | Repo deltas (`git fetch` on in-scope repos), PR head, Jira ticket |
| **Injected** (run start) | Layer-3 agent definition, credentials (GitHub/Jira tokens), model config — **never baked** |

Principles:

- **The image is a cache, not a snapshot to trust.** Runtime always
  fetches in-scope repos to current — incremental against a baked
  clone, so seconds not minutes. Rebuild cadence (weekly is fine) is
  therefore a latency optimization, never a correctness concern;
  `review-state.yaml` records the exact SHAs reviewed.
- **One image, two consumers.** The same image serves as a
  devcontainer for interactive human use of the preset and as the
  agent runtime. Deployment preserves the layer separation, and human
  use continuously validates the image.
- **Bake validation**: the image build runs a smoke test (pipeline
  dry-run against a fixture PR) so a broken plugin pin fails the
  build, not a live review.

Build sketch: base (Claude Code CLI, git, yq) →
`setup.sh init tnf-reviewer && setup.sh clone` (the large cached
layer) → install skill manifest → smoke test. Scheduled rebuild;
entrypoint script does the fetch/inject steps then starts the agent.

### State persistence across ephemeral runs

Delta re-review needs the previous run's `review-state.yaml`, but each
agent run is a fresh container — "disk is the message bus" must extend
to "and the disk survives the container". Decision: the project
workspace round-trips through a **dedicated git branch** (e.g. a
`reviews` branch on the operator's dev-env fork, one directory per
PR). Bootstrap pulls it; teardown pushes it. This keeps the pipeline
runtime-agnostic (GitHub Action, OpenShift CronJob/Job, local podman)
and makes the audit trail version-controlled. A persistent volume
works too but ties the pipeline to one platform.

## State model

A pipeline run lives in a project workspace
(`projects/pr-<repo>-<number>/`) containing:

| File | Role |
|------|------|
| `CLAUDE.md` | Lean index (existing convention) — frontmatter carries PR URL, reviewed SHA, repos, worktrees |
| `review-state.yaml` | **The bus.** Findings with stable IDs, stage status, verdicts, posting state. Schema: `review-state.schema.yaml` |
| `intent.md` | Snapshot of the Jira ticket: description, acceptance criteria, linked enhancement |
| `findings.md` | Human-readable rendering of surviving findings (regenerated from review-state.yaml) |

Each finding carries a stable ID (`F-001`, …), a diff anchor
(file, line, SHA), category, severity, and accumulates fields as it
moves through stages: `vet.verdict` + rationale, posting decision,
GitHub comment ID. See the schema file for the full annotated format.

## Pipeline stages

```
intake → code review (fan-out) → vet (autonomous)
      → spec compliance ─┐
      → cross-repo check ─┼→ compose pending review → HUMAN SUBMITS
      → arch-fit (grounded) ─┘
                          ↑
            delta re-review on new push
```

### 1. Intake (orchestrator, main thread)

Input: a PR URL. Actions:

- Run the non-interactive path of `/project:new` (analysis type with
  PR URL — already creates the `pr/<number>` worktree). Defaults are
  taken silently; nothing prompts.
- Fetch the linked Jira (from the PR description or commit messages)
  and snapshot it into `intent.md`. Jira access is via MCP server or a
  fetch script — an explicit dependency to resolve in Phase 2.
- Initialize `review-state.yaml` with PR metadata and head SHA.
- Determine relevant repos from the diff + TNF context and record them
  in project frontmatter (drives lazy context loading later).

### 2. Code review fan-out

Run the existing review plugin (e.g. the Anthropic code-review plugin)
against the PR worktree — unchanged. A thin **harvest step** then
parses its output into `review-state.yaml` entries
(`source: code-review`, `vet.verdict: pending`). The plugin is treated
as a black box; only the harvest step knows its output format.

### 3. Autonomous vet

`/vet-review` in **pipeline mode** (see SKILL.md): loads the project
context — `intent.md`, the touched repo's `TNF-CONTEXT.md`/`CLAUDE.md`,
`presets/tnf/context.md` — then vets each pending finding against the
diff and surrounding code, writing `confirmed`/`dropped` verdicts with
rationale into `review-state.yaml`. No interactive gate. The
interactive one-by-one mode remains the default for manual use.

### 4. Context passes (parallel subagents)

Three passes the code-review plugin structurally cannot do, each
appending findings to `review-state.yaml` with its own `source` tag:

- **Spec compliance** (`source: spec-compliance`): diff vs. `intent.md`
  acceptance criteria vs. the enhancement doc in `repos/enhancements/`.
  Flags missing scope, symptom-vs-ticket fixes, and untouched lifecycle
  phases (e.g. change works in steady-state, bootstrap→handover path
  unverified).
- **Cross-repo contract check** (`source: cross-repo`): grep the
  *other* TNF repos for consumers of whatever the PR touches — env var
  names, file paths, pacemaker resource names, CLI flags. Flags
  contract breaks between CEO, MCO, resource-agents, fence-agents.
- **Architecture fit** (`source: arch-fit`): grounded, not free-floating.
  Find 2–3 merged changes that touched the same paths
  (`git log` — blobless clones keep full history), compare the PR
  against those precedents plus the repo's TNF-CONTEXT.md. Output only
  pattern mismatches with a cited precedent.

New findings from these passes also get vetted (a second, short vet
round) before composition.

### 5. Compose & post

From surviving findings, compose a **pending** GitHub review:

- Inline comments for `critical`/`major` findings (anchored via the
  stored diff positions); `minor`/clarity findings collapse into the
  top-level summary to keep the PR uncluttered.
- Top-level comment: verdict recommendation, summary table, and a
  "dropped as noise" table for transparency.
- Record GitHub comment IDs back into `review-state.yaml`.

**Verdict policy** (mechanical, not vibes — this table is layer-3
policy, applied only in autonomous runs; interactive users decide
their own verdict):

| Surviving findings | Recommendation |
|--------------------|----------------|
| Any bug / security / contract-break | REQUEST_CHANGES |
| Only clarity / minor / pattern notes | COMMENT |
| None | COMMENT ("no issues found" summary) |
| — | Never APPROVE (Prow gating; human-only) |

### 6. Posting modes (human gate only in hold mode)

Two posting modes, selected per run:

- **`auto` (the end state):** the pipeline submits the review itself —
  REQUEST_CHANGES or COMMENT per the verdict policy, never APPROVE.
  Fully unattended; no human interaction.
- **`hold` (evaluation/development):** the pipeline stops after
  compose. A human skims the composed review, deletes or edits
  anything, and posts it — one interaction instead of one per finding.
  Hold earns the trust to flip to `auto`.

### 7. Delta re-review on push

PRs get force-pushed mid-review. On a new head:

- Record the new SHA; diff `old-head..new-head`.
- Re-anchor findings: unchanged code keeps its verdict; findings whose
  anchors moved or whose code changed go back to `vet.verdict: pending`
  and are marked `stale` pending re-vet.
- Never repost a finding that has a `posting.comment_id`. Resolved
  findings (code now fixed) get marked `resolved`.
- Only the delta goes through stages 2–4.

Without this, re-runs spam duplicate comments and the pipeline becomes
unusable within a week.

## Orchestration

A `/pr-review:run <PR-URL>` skill (layer 2 — shipped by the
`tnf-reviewer` preset) chains the stages, fanning out subagents (Task
tool) for stages 2–4 and the vet. Subagents never talk to each other —
`review-state.yaml` is the only contract. The orchestrator updates
stage status in the state file so an interrupted run is resumable via
`/project:resume`.

The skill itself stays mode-neutral: invoked by a human, it honors
interactive gates; invoked under the agent definition (layer 3), gates
collapse per that file's policy. The agent definition is the only
place that decides between the two.

PR-event automation (auto re-review on push via PR subscriptions or a
polling loop) is deliberately last — the pipeline must be solid when
invoked manually before it runs unattended.

## Phasing

| Phase | Deliverables |
|-------|-------------|
| 1 | `review-state.yaml` schema; vet-review pipeline mode; manual orchestration (human invokes each stage) |
| 2 | Preset mechanics: `extends:` composition, skill-manifest resolution; carve out the `tnf-reviewer` preset (layer 2) |
| 3 | Intake automation: non-interactive `/project:new` path, Jira snapshot (`intent.md`), harvest step for the review plugin output |
| 4 | Context passes: spec compliance, cross-repo contract check, grounded arch-fit — parameterized via `preset.yaml` `review:` block |
| 5 | Compose/post pending review + verdict policy; delta re-review |
| 6 | `/pr-review:run` orchestrator skill; agent definition (layer 3, `agents/reviewer.md`); optional PR-event automation |
| 7 | Runtime image: Containerfile + scheduled build with bake validation; bootstrap/teardown scripts (delta fetch, state-branch round-trip, credential injection) |

## Open questions

- **Jira access**: MCP server vs. REST script; where credentials live.
- **Harvest format coupling**: the parse step depends on the review
  plugin's output format — pin a plugin version or make harvest robust
  to prose.
- **Preset composition semantics**: what `extends:` merges vs.
  overrides (repo lists, context files, settings templates) — needs a
  decision before external preset packs ship.
- **Agent definition format**: plain operating-manual markdown vs. a
  Claude Code native subagent definition in `.claude/agents/` — the
  latter makes the whole reviewer spawnable as a subagent.
