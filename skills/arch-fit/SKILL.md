---
name: arch-fit
description: "Review pass — judge a PR's architecture against 2–3 merged precedents on the same paths instead of free-floating opinions. STUB."
user-invocable: false
---

# arch-fit review pass

> **STUB (Phase 4).** Not yet implemented.

## Intended responsibility

Ground architecture feedback in **precedent**, not taste:

- Find **2–3 recently merged PRs** touching the same files / packages.
- Compare the PR's structure, naming, and patterns against those precedents.
- Flag deviations that would make the change an outlier on those paths;
  cite the precedent PR as evidence.

This keeps the pass from producing free-floating architecture opinions that
reviewers rightly ignore.

Appends findings to `review-state.yaml` with `source: arch-fit`.
