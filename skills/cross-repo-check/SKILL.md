---
name: cross-repo-check
description: "Review pass — grep the other TNF repos for consumers of what the PR touches (CEO ↔ MCO ↔ resource-agents ↔ fence-agents) and flag contract breaks. STUB."
user-invocable: false
---

# cross-repo-check review pass

> **STUB (Phase 4).** Not yet implemented. Repo set comes from
> `preset.yaml` (`review.contract_check_repos`).

## Intended responsibility

Catch breakage a standalone single-repo review structurally cannot:

- For each symbol / interface / file the PR changes, **grep the consumer
  repos** for callers and dependents.
- Flag **contract breaks** across the TNF component boundaries
  (cluster-etcd-operator ↔ machine-config-operator ↔ resource-agents ↔
  fence-agents ↔ api).
- Report the consuming site (`file:line` in the other repo) as the finding
  anchor.

Appends findings to `review-state.yaml` with `source: cross-repo`.
