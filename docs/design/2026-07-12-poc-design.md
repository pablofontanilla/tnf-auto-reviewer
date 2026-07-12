# TNF Auto-Reviewer PoC — Design

**Date:** 2026-07-12
**Status:** Approved (brainstorm complete)
**Supersedes for PoC scope:** the phasing in `agent-pr-review.md` (which
remains the full-vision reference — layering rationale, generalization
mechanism, distribution model)

## Summary

A proof of concept of the agent-based TNF PR-review pipeline: a
self-contained Claude Code plugin plus a podman container image that
reviews a TNF PR **unattended**. The end state (`auto` mode) posts the
review to the PR itself, CodeRabbit-style — no human in the loop (the
agent still never approves). During evaluation and development the
pipeline runs in `hold` mode: all stages run, but the composed review is
held for a human to skim and post. Built as a **walking skeleton**: the
thinnest end-to-end unattended run first, then review-quality passes,
then hardening.

## Goals

- Unattended run against a live TNF PR producing a review good enough to
  post as-is. In the end state (`auto`) the pipeline posts it itself;
  there is no human gate.
- **Hold-before-post (evaluation/development)**: the pipeline can run
  fully but stop before posting anything to GitHub; a human resumes later
  in an ad-hoc Claude session to post verbatim or adjust and re-run. Hold
  exists to build trust toward `auto`, not as the final workflow.
- Delta re-review on force-push without duplicating comments (M3).

## Non-goals (PoC)

- Generalization beyond TNF — domain preset, repo lists, and skills are
  hardcoded; refactor to a generic reviewer later.
- Workspace-plugin preset mechanics (`extends:`, skill manifests) — the
  plugin here is self-contained.
- PR-event automation (auto-trigger on push), GitHub Action / OpenShift
  runtimes, service accounts, git-branch state round-trip, scheduled image
  rebuilds — all post-PoC hardening.
- Agent ever approving a PR. Never. Approval stays human (Prow gating).

## Decisions

| Topic | Decision |
|-------|----------|
| v0 bar | Unattended agent (not manual stage-driving) |
| Dependencies | Hardcode TNF specifics; self-contained plugin |
| Review engine | Built-in `/code-review` skill, harvested into state |
| Jira access | REST script + PAT (issues.redhat.com), runtime-injected |
| Runtime | Local podman, triggered by hand (cron later) |
| State store | Bind-mounted host dir (`~/tnf-reviews/`) |
| Orchestrator | Claude Code **dynamic workflow** (JS, `.claude/workflows/`) |
| Posting mode | **Workflow input parameter** (`--posting-mode=hold\|auto`, container env → workflow arg, recorded in state). Default `hold` during PoC; `auto` (fully unattended posting) is the end state |
| Acceptance test | Live TNF PR: unattended run → skim → post (M1); push → no dupes (completes at M3) |

## Architecture

### Runtime shape

```
host machine
├── ~/tnf-reviews/                      ← bind mount: durable state
│   └── pr-<repo>-<number>/
│       ├── review-state.yaml           ← THE bus (survives container death)
│       ├── intent.md                   ← Jira snapshot (M2)
│       ├── findings.md                 ← human-readable rendering
│       └── CLAUDE.md                   ← lean index for ad-hoc sessions
└── podman run --rm -v ~/tnf-reviews:/state -e GITHUB_TOKEN -e JIRA_PAT
    -e POSTING_MODE=hold|auto ...
    └── container (Containerfile in this repo)
        ├── baked: TNF repos (blobless clones), TNF context files,
        │         this plugin, Claude Code CLI
        ├── injected: credentials, model config     ← never baked
        └── entrypoint: fetch in-scope repos → claude -p (headless)
            └── /tnf-reviewer:run <PR-URL> → dynamic workflow
```

One container run = one pipeline pass over one PR. Same state dir + new
head = delta re-review.

### Layering (PoC-flattened)

The three-layer model from `agent-pr-review.md` survives conceptually but
ships in this one repo, hardcoded to TNF:

| Layer | Here | Content |
|-------|------|---------|
| 1 — domain | baked image layer | TNF repos + context files |
| 2 — capability | `skills/`, `.claude/workflows/`, schema | procedure: stages, workflow, state contract |
| 3 — autonomy | `agents/reviewer.md` | judgment policy applied inside stages: verdict, escalation, never-approve |

### Orchestrator: dynamic workflow

`.claude/workflows/review.js` sequences the stages using `agent()` (fresh
context per stage) and `parallel()` (context passes). Rationale:
deterministic sequencing, schema-validated stage outputs, isolated
contexts, headless-capable — replaces prose Task-tool chaining. It takes
`posting_mode` as an input parameter and records it in state.

The workflow and the agent definition are **complementary, not
overlapping**: the workflow (layer 2) owns procedure — stage order,
parallelism, idempotent skipping, the posting-mode branch — while
`agents/reviewer.md` (layer 3) owns the judgment policy the stage agents
run under (verdict, escalation, never-approve). The workflow script
absorbs what "gate collapsing" used to mean under prose orchestration.

Workflow session-only resume is irrelevant because the script is
**idempotent over `review-state.yaml`**: on start it reads `stages:` and
skips anything `done`. Crash-restart, `hold` stop, and ad-hoc resume all
re-enter identically. `/tnf-reviewer:run` remains the thin entrypoint that
launches the workflow; the workflow returns a run summary (stage statuses,
finding counts, verdict recommendation) as the container's output.

## Pipeline

```
1 intake ──► 2 code-review + harvest ──► 3 vet (round 1)
                                              │
                  ┌──────────── parallel() ───┤
                  │ 4a spec-compliance        │        (M2)
                  │ 4b cross-repo check       │
                  │ 4c arch-fit (grounded)    │
                  └───────────┬───────────────┘
                       5 vet (round 2, new findings only)
                              │
                       6 compose ──► composed review in state
                              │
              ┌── posting_mode? ──┐   (workflow input param)
            auto                hold
              │                   │
       7 post & submit       STOP. stages.post = held
       review (verbatim           │
       from state; COMMENT   ad-hoc Claude session later:
       or REQUEST_CHANGES,        ├─► post verbatim (stage 7 as-is,
       never APPROVE)             │   human skims first)
              │                   └─► adjust findings → re-run 2–6
              ▼
       done — no human step
```

Stage notes:

- **Intake** builds a context-ful workspace-project environment: project
  dir, PR worktree, and links to each relevant repo's `CLAUDE.md` +
  supplemental TNF context files (`TNF-CONTEXT.md`/`DOMAIN-CONTEXT.md`).
  Only repos deemed relevant from the PR (diff + description) are linked
  in; `repos_in_scope` recorded in state. **The review sees the domain,
  not just the diff.**
- **Code review + harvest**: run built-in `/code-review` in that
  environment; a thin harvest step parses its output into findings
  (`source: code-review`, `vet.verdict: pending`). Only the harvest step
  knows the output format; fixtures pin it (see Testing).
- **Vet** (pipeline mode, no interactive gate) writes
  `confirmed`/`dropped` + rationale per finding. Round 2 vets only
  `pending` findings from the context passes.
- **Context passes (M2)** append findings with their own `source` tags:
  spec-compliance (diff vs `intent.md` acceptance criteria vs enhancement
  doc; flags missing scope and untouched lifecycle phases), cross-repo
  (grep consumer repos for contract breaks), arch-fit (compare against
  2–3 merged precedents on the same paths, cited).
- **Compose** builds the full review in state: inline comments for
  critical/major, minors collapsed into the top-level summary, verdict
  recommendation, dropped-as-noise table. Regenerates `findings.md`.
- **Post** writes the composed review to GitHub **verbatim from state** —
  deliberately trivial so what sits in state is exactly what lands. In
  `auto` mode it submits the review directly (COMMENT or REQUEST_CHANGES
  per verdict policy, never APPROVE); in `hold` mode the same stage runs
  later from the ad-hoc session after a human skim. Records comment IDs
  back into state.

## State model

Schema: `review-state.schema.yaml` (reconstructed; finalized in M1).
Additions from this design:

- `posting_mode: auto | hold` (run-level; set from the workflow input
  parameter, recorded in state for audit)
- stage status enum gains **`held`** ("deliberately stopped, awaiting
  human resume" — distinct from `pending`)
- `stages.post` split from `stages.compose`

Finding lifecycle: `active` → `stale` (anchor/code moved on new push →
verdict reset to `pending`, re-vetted) → `resolved` (fixed; never
repost). A finding with `posting.comment_id` is **never reposted** — the
idempotency key against comment spam.

Delta re-review (M3): new run, same state dir; intake sees
`head_sha ≠ reviewed_sha`, diffs old..new, re-anchors, marks stale; only
the delta flows through stages 2–6.

The state dir is the PoC audit trail: verdicts with rationale, dropped
table, exact SHAs.

## Runtime image

| Layer | Content |
|-------|---------|
| Base | Claude Code CLI, git, `yq`/`jq`, python3 |
| Baked (big cached layer) | Blobless TNF repo clones (full history for arch-fit), TNF context files, this plugin |
| Bake validation (M3) | Build-time smoke test: dry-run against a fixture; broken pin fails the build |
| Never baked | Credentials, model config, PR/state under review |

Entrypoint: `git fetch` in-scope repos (image is a cache, not a snapshot
— correctness never depends on rebuild cadence; state records exact SHAs)
→ resolve state dir → `claude -p` headless with pre-approved tools →
exit code + run summary.

Auth (PoC = personal credentials): `GITHUB_TOKEN` (gh CLI posts the
review), `JIRA_PAT` (fetch script only), Claude auth per headless
conventions. Image rebuilds manual.

## Error handling & escalation (layer 3)

- Stage failure → stage marked `failed` + notes, run stops, non-zero
  exit. **Never post a partial review.** Re-run re-enters at the failed
  stage.
- Escalate by stopping, not guessing: Jira unresolvable →
  spec-compliance runs degraded (PR description + enhancement doc) and
  says so; ambiguous re-anchor after force-push → finding marked `stale`
  for human eyes, not dropped; fencing/STONITH changes without precedent
  cover → flagged prominently in the summary.
- `hold` doubles as the trust valve: escalation-worthy runs land as
  `held` even in `auto` mode.

## Testing

- **Schema/unit:** validate state instances against the schema; harvest
  parsing tested against captured `/code-review` output fixtures (pins
  the format-coupling risk).
- **Stage:** every stage skill runnable standalone in an interactive
  session against a fixture state dir — also the manual-debugging path.
- **End-to-end:** replay a merged TNF PR as repeatable pre-flight; then
  the acceptance test — live PR, `hold`, skim `findings.md`, post
  verbatim, push a commit, verify update-without-duplication.

## Milestones

| Milestone | Delivers | Proves |
|-----------|----------|--------|
| **M1 — Skeleton** | Schema finalized; intake (context-ful env, no Jira); `/code-review` + harvest; vet round 1; compose; post-verbatim; workflow script; Containerfile; `hold` default | The unattended loop end-to-end: container → held review → ad-hoc resume → posted |
| **M2 — Signal** | Jira REST script + `intent.md`; spec-compliance, cross-repo, arch-fit in `parallel()`; vet round 2 | The differentiated review quality |
| **M3 — Hardening** | Delta re-review; verdict policy + escalation wired into agent def; bake validation; `auto` mode earned | Unattended on live PRs without babysitting |

M1 deliberately includes the container + workflow (the integration risk)
and excludes everything skippable. Delta re-review sits in M3 because
`hold` + fresh runs cover until then; pull it forward if force-pushes
bite earlier.
