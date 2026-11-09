# ADR 0010: The platform team owns defaults; packages come in two tiers and two verbs

**Status:** Proposed · 2026-11-08
**Supersedes / relates to:** docs/debate-review.md §9; docs/stories.md story 7

## Context

Golden paths (which CA, which backup target, whose pager) are org-specific. Operators and CRDs are cluster-scoped and need cluster-admin. Helm does not check dependencies against the cluster, and uninstall leaves claims behind. Adding a command to the toolbox is an image operation, not a Helm release.

## Decision

Platform packages (operators, controllers, CRDs) are installed by the platform team through GitOps and appear in `caps`. User packages are namespaced releases of pre-approved charts via `pkg install`, reconciled by Argo CD or Flux. `tool add` adds a command to the toolbox as an image layer and requires an annotation entry. Every package carries a manifest: provides, requires (checked against `caps`), creates, cost estimate, retains, signature.

## Consequences

Users see which default came from which profile through `explain`. `pkg remove` prints what is retained and how to destroy it. Drift is visible because installs are Git-backed.

## Amendment 1 · 2026-11-09 · managed objects are written as intent (user stories US-03 and US-09, L3.2 and L9.3)

**Context.** Walkthrough gap G3 made `release` write the image version to the GitOps repository. Everything else that mutates an object GitOps owns, a canary route weight, a replica count, a rollout-restart annotation, still patched the object directly, and self-heal reverts each within minutes. The shell is imperative and the platform is declarative, and the seam has to be in one place.

**Decision.** Every mutation of an object under GitOps management is written as intent. Management is detected from the tracking label or annotation Argo CD or Flux sets. The session writes the change to the repository and branch the platform's `release_path` names, waits for reconciliation, and records the commit and the resulting object hash in the receipt. This covers `release`, `scale`, route weights, `kill -HUP` under `reload: restart` (a version annotation bump), `mkdir` of a managed topic or namespace, and `chmod` where the ACL object is managed. `explain` prints `managed-by: gitops → via commit to <repo>@<branch>` for every such step. A direct patch to a managed object is refused with exit 3 and the intent spelling; `release_path=direct` is allowed only for namespaces the platform marks as development.

**Consequences.** The repository must be a device the session can write (ADR 0011 amendment 2). Reconciliation time is part of every unit verb's latency and is printed. Residency and other defaults that the platform sets per namespace (ADR 0011 amendment 3) travel the same way: they are profile values the platform commits, not flags users remember.
