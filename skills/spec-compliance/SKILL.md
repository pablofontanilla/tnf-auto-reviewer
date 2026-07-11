---
name: spec-compliance
description: "Review pass — check a PR diff against its Jira acceptance criteria and the TNF enhancement doc; flag half-implemented scope and steady-state-only fixes. STUB."
user-invocable: false
---

# spec-compliance review pass

> **STUB (Phase 4).** Not yet implemented. Parameterized via `preset.yaml`
> (`review.lifecycle_phases`, `review.tracker`).

## Intended responsibility

Compare the diff against the **intended scope**, not just the code:

- Diff vs. **Jira acceptance criteria** — is every criterion addressed?
- Diff vs. the **TNF enhancement doc** — does the change match the approved
  design?
- Flag **half-implemented scope** (criteria with no corresponding change).
- Flag **steady-state-only fixes** that ignore the other TNF lifecycle
  phases (`install` / `bootstrap` / `handover`) when the change should span
  them.

Appends findings to `review-state.yaml` with `source: spec-compliance`.
