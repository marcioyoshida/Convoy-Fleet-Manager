# ADR 0010: The platform team owns defaults; packages come in two tiers and two verbs

**Status:** Proposed · 2026-11-08
**Supersedes / relates to:** docs/debate-review.md §9; docs/stories.md story 7

## Context

Golden paths (which CA, which backup target, whose pager) are org-specific. Operators and CRDs are cluster-scoped and need cluster-admin. Helm does not check dependencies against the cluster, and uninstall leaves claims behind. Adding a command to the toolbox is an image operation, not a Helm release.

## Decision

Platform packages (operators, controllers, CRDs) are installed by the platform team through GitOps and appear in `caps`. User packages are namespaced releases of pre-approved charts via `pkg install`, reconciled by Argo CD or Flux. `tool add` adds a command to the toolbox as an image layer and requires an annotation entry. Every package carries a manifest: provides, requires (checked against `caps`), creates, cost estimate, retains, signature.

## Consequences

Users see which default came from which profile through `explain`. `pkg remove` prints what is retained and how to destroy it. Drift is visible because installs are Git-backed.
