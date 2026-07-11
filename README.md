# tnf-auto-reviewer

An **agent-based PR-review pipeline** for the TNF (Two Nodes with Fencing)
OpenShift topology. It turns the manual TNF review workflow into a pipeline
whose only human touchpoint is skimming and submitting **one pending GitHub
review**.

> **Status:** scaffolding. Design is settled; implementation follows the
> phase plan in `docs/design/agent-pr-review.md`. Skill and agent files here
> are stubs marking intent, not yet working implementations.

## How it works

Stages communicate **exclusively through an on-disk `review-state.yaml`
artifact** (findings with stable IDs, diff anchors, vet verdicts, posting
state). No stage depends on shared conversation context, so every stage runs
as a fresh, resumable subagent — an interrupted run picks up where it left
off.

The mechanical verdict policy recommends; **the agent never approves**
(Prow gating keeps approval human).

## Three layers

| Layer | Artifact | Responsibility |
|-------|----------|----------------|
| 1 — domain | `tnf` preset (in the `workspace` plugin) | repos, context, docs — "what the world is" (reference; not shipped here) |
| 2 — workflow | `preset.yaml` (`tnf-reviewer`, `extends: tnf`) | review skills + state schema + review params (contract-check repos, lifecycle phases, tracker). Usable by a human with interactive gates on. |
| 3 — autonomy | `agents/reviewer.md` | pure autonomy policy: gate collapsing, verdict policy, never-approve, escalation, termination. No procedure, no domain knowledge. |

Pattern: **preset = capability, agent definition = autonomy.** A future
`tnf-bug-triage` agent would be a new layer-3 file over the same layer-1/2.

## The differentiating review passes

- **Spec compliance** — diff vs. Jira acceptance criteria vs. enhancement
  doc; catches half-implemented scope and steady-state-only fixes.
- **Cross-repo contract check** — greps consumer TNF repos (CEO ↔ MCO ↔
  resource-agents ↔ fence-agents) for breaks. Structurally impossible for a
  standalone review plugin.
- **Grounded architecture fit** — compares against 2–3 merged precedents on
  the same paths instead of free-floating opinions.

## Layout

```
.claude-plugin/plugin.json    plugin manifest
preset.yaml                   tnf-reviewer workflow preset (extends: tnf)
skills/
  run/                        /tnf-reviewer:run — orchestrator (entrypoint)
  spec-compliance/            diff vs Jira AC vs enhancement doc
  cross-repo-check/           consumer-repo contract check
  arch-fit/                   grounded architecture-fit pass
agents/reviewer.md            layer-3 autonomy policy
docs/design/
  agent-pr-review.md          full design doc (source of truth)
  review-state.schema.yaml    state-artifact schema
```

## Relationship to the `workspace` plugin

The [`workspace`](https://github.com/fonta-rh/multi-repo-dev-env) plugin is a
**consumer/dependency** of this repo, not where this code is authored. Preset
mechanics this pipeline needs — preset `extends:` composition and
preset-shipped `skills/`+`agents/` — are extensions tracked against that
plugin.

## License

Apache-2.0
