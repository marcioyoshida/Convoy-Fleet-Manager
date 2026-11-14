# ADR 0010: The platform team owns defaults; packages come in two tiers and two verbs

**Status:** Proposed · 2026-11-08 · amendment 1 revised 2026-11-12 with user-story finding L27.4 (marked inline)
**Supersedes / relates to:** docs/debate-review.md §9; docs/stories.md story 7

## Context

Golden paths (which CA, which backup target, whose pager) are org-specific. Operators and CRDs are cluster-scoped and need cluster-admin. Helm does not check dependencies against the cluster, and uninstall leaves claims behind. Adding a command to the toolbox is an image operation, not a Helm release.

## Decision

Platform packages (operators, controllers, CRDs) are installed by the platform team through GitOps and appear in `caps`. User packages are namespaced releases of pre-approved charts via `pkg install`, reconciled by Argo CD or Flux. `tool add` adds a command to the toolbox as an image layer and requires an annotation entry. Every package carries a manifest: provides, requires (checked against `caps`), creates, cost estimate, retains, signature.

## Consequences

Users see which default came from which profile through `explain`. `pkg remove` prints what is retained and how to destroy it. Drift is visible because installs are Git-backed.


Note · 2026-11-12: the draft [ADR 0024](0024-a-device-is-installed-by-terraform.md) adds a third kind of package content, the device package's contract and its Terraform modules per provider, installed by `mknod` through the platform's runner; day-2 workloads stay with GitOps as decided here.
## Amendment 1 · 2026-11-09 · managed objects are written as intent (user stories US-03 and US-09, L3.2 and L9.3)

**Context.** Walkthrough gap G3 made `release` write the image version to the GitOps repository. Everything else that mutates an object GitOps owns, a canary route weight, a replica count, a rollout-restart annotation, still patched the object directly, and self-heal reverts each within minutes. The shell is imperative and the platform is declarative, and the seam has to be in one place.

**Decision.** Every mutation of an object under GitOps management is written as intent. Management is detected from the tracking label or annotation Argo CD or Flux sets. The session writes the change to the repository and branch the platform's `release_path` names, waits for reconciliation, and records the commit and the resulting object hash in the receipt. This covers `release`, `scale`, route weights, `kill -HUP` under `reload: restart` (a version annotation bump), `mkdir` of a managed topic or namespace, and `chmod` where the ACL object is managed. `explain` prints `managed-by: gitops → via commit to <repo>@<branch>` for every such step. A direct patch to a managed object is refused with exit 3 and the intent spelling; `release_path=direct` is allowed only for namespaces the platform marks as development. Resources outside the clusters that a Terraform state manages are managed objects too: a write to one (a bucket policy, a topic's config, a DNS record) is written as intent to the IaC repository, as a pull request against the module instance that declares it, and waits for the platform's runner to apply; under `iac=refuse` it is refused, and `explain` prints the resource address (revised 2026-11-12, L27.4; the module instances are ADR 0024's).

**Consequences.** The repository must be a device the session can write (ADR 0011 amendment 2). Reconciliation time is part of every unit verb's latency and is printed. Residency and other defaults that the platform sets per namespace (ADR 0011 amendment 3) travel the same way: they are profile values the platform commits, not flags users remember.

## Rules · 2026-11-09 · from the user-story probes

Each line is a `rule` finding from `docs/user-stories/README.md` written here as decided; the story holds the reasoning.

- **L2.2** (US-02) bands for executables are tagged `declared` in `explain`; the receipt records declared versus observed; a package may provide an estimator hook that reads the job config.
- **L4.3** (US-04) platform default `idle_timeout` per unit class with a GPU; idle is scale-to-zero with volumes retained (ADR 0006); `ps` shows IDLE and the receipt the idle cost.
- **L7.2** (US-07) metered entries are labelled `declared`; a later `settled` source (provider billing export) reconciles; FinOps sees both; a package whose declared spend is under settled by a platform threshold is flagged in `pkg list`.
- **L7.6** (US-07) `egress:` with hostnames requires `caps: fqdn-policy`; without it the planner refuses or routes through the platform's egress proxy package; `explain` names the mechanism.
- **L12.3** (US-12) consumer units default to `maxSurge: 0`, the cooperative-sticky assignor and a drain on TERM that commits before leaving; printed by `explain`.
- **L16.4** (US-16) `managed-by` under the node names the manager and its object; `rm` on a node is written as intent to that object (ADR 0010 a1), fencing follows the instance's power-off, and the replacement is the manager's; with no manager `rm` is refused and names the `page:` route for the hardware owner.
- **L16.5** (US-16) every `health` source is a capability (`dcgm`, `node-agent`); an absent source reads `unknown`, never `ok`, and `explain` on a GPU plan prints `gpu health: unknown` when `dcgm` is missing.
- **L25.3, L25.6, L25.8, L25.9, L25.12** (US-25) new platform defaults for the inner loop of ADR 0022: `lend_exclude` and `lend_data`, `debug_pause_max`, `follow_rebuild_max`, `build_egress` and `dev_mounts`, each printed by `explain` where it applies (full rules in ADR 0022).
- **L27.15** (US-27) the price files of every provider the platform serves are platform defaults, so `cost --reprice-at` and a script's horizon band can price a target before anything moves (full rule in ADR 0013).
- **L28.8, L28.13** (US-28) device packages band usage-priced resources from contract inputs, and pin module versions per instance so a package upgrade never re-plans instances (full rules in ADR 0024).
- **L29.6, L29.9** (US-29) each computer declares its price model; `release` and `scale` on a module-installed computer write module inputs as intent (full rules in [ADR 0025](0025-a-computer-is-a-device-kubernetes-is-one-driver.md)).
- **L30.2, L30.17** (US-30) a tag that GitOps, CI or a replication rule owns is moved only as intent to its owner; production uses public images only by digest from a platform cache or mirror (full rules in [ADR 0026](0026-a-registry-is-a-device-a-tag-is-a-name-for-a-digest.md)).
- **L31.2, L31.6, L31.15** (US-31) routes never edit schedules or alert rules, and a route kept in GitOps is changed as intent; `notify_max`, `page_before`, `expiry_warn` and working hours are platform defaults (full rules in [ADR 0027](0027-convoy-tells-a-group-about-its-own-objects.md)).
