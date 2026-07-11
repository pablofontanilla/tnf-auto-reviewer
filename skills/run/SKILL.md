---
name: run
description: "Orchestrate the TNF PR-review pipeline — intake a PR, run each review pass into review-state.yaml, vet findings, compose and post one pending GitHub review. STUB."
argument-hint: "[PR number | PR URL | owner/repo#number]"
user-invocable: true
---

# /tnf-reviewer:run — pipeline orchestrator

> **STUB (Phase 6).** This is the layer-3 entrypoint that sequences the whole
> pipeline. Not yet implemented. See `docs/design/agent-pr-review.md` and
> `agents/reviewer.md`.

## Intended responsibility

Drive the full review as a sequence of **fresh subagents** that communicate
only through the on-disk `review-state.yaml` artifact:

1. **Intake** — resolve the PR, snapshot the diff + Jira acceptance criteria,
   seed `review-state.yaml`.
2. **Review passes** — dispatch each pass (code-review harvest,
   `spec-compliance`, `cross-repo-check`, `arch-fit`); each appends findings
   with stable IDs and diff anchors.
3. **Vet** — adversarial second pass filters noise; writes `vet.verdict` +
   rationale per finding.
4. **Compose & post** — build a pending GitHub review (inline critical/major
   comments + top-level summary with verdict recommendation + dropped-findings
   table). Record `posting.comment_id`s. **Never approve.**
5. **Delta re-review** — on a new push, re-anchor findings to the new SHA and
   re-vet only changed code; never duplicate posted comments.

Each step is resumable: an interrupted run continues from the state file via
`/workspace:resume`.
