# ADR 0007: Secrets are references in the session, never values

**Status:** Proposed · 2026-11-08
**Supersedes / relates to:** docs/debate-review.md §3

## Context

A note mapped secrets to env vars, proposed scrubbing them from history, and claimed External Secrets Operator avoids etcd. ESO writes Secrets into etcd; env vars leak into child processes, dumps and logs; a value that reaches the client has already leaked.

## Decision

`secret set NAME` writes server-side from a prompt or stdin. In commands, `$NAME` compiles to a `secretKeyRef` or a Secrets Store CSI mount at `/run/secrets/NAME`; env delivery is opt-in with a warning. Users get `create pods` without `get secrets` (reference-only RBAC). Cloud credentials come from workload identity with projected tokens, not stored secrets.

## Consequences

Nothing to scrub. Attribution of secret use is emitted by the session at launch, since the API audit log attributes the read to the kubelet. Platform must provide etcd KMS encryption and the CSI driver; both appear in `caps`.

## Amendment 1 · 2026-11-08 · versions, references and rotation (walkthrough 9)

**Context.** A reference-only model needs an answer to "who uses this secret right now?" and "how do I replace it everywhere without handling the value?". The decision above gives neither.

**Decision.** Secrets are versioned at the vault. The session maintains a reference index per cluster: which process mounts which secret at which version, from the mounts it created plus the fleet inventory for processes created elsewhere. `fuser /secrets/<name> [--version=vN] [--cluster=…|all]` reads it, partial and degraded on unreachable clusters (ADR 0011). `secret rotate <name>` writes a new version and retains the old; `secret retire <name> vN` is the destructive step, refused while any reachable reference exists and deferrable with `--when` until the condition is provably true. No verb ever prints a value.

**Consequences.** Rotation is a loop of `kill -HUP` per cluster (ADR 0008 amendment 2) and a retire. The index is only as complete as the inventory; a cluster the inventory cannot see is reported, never assumed empty. See docs/walkthroughs.md, gap G10.

## Amendment 2 · 2026-11-09 · the artifact store is a secret sink (user story US-07, L7.1)

**Context.** References protect the session, not what a process does once it holds the value: a process with a mounted secret can print it to stdout, which becomes a retained artifact, or to stderr, which becomes a retained log. Both outlive the secret's version.

**Decision.** Artifacts and logs of a process that mounted any secret carry the retention class `sensitive` by default, which the platform maps to a shorter retention and a narrower reader set. The artifact writer (`convoy-put`) and the log shipper run secret-pattern detection using the vault's value hashes and shape rules; a match quarantines the artifact or log segment under class `sensitive-quarantine`, fails the stage with exit 1 and names the secret in the receipt without its value. Detection is best effort and the receipt says `scrub: best-effort, ruleset vN`. The same scrub applies to any context that leaves the session for a model (ADR 0014 decision 5).

**Consequences.** A false positive fails a stage, which is the cheaper error. The vault must expose value hashes to the scrubber, which is a platform capability (`caps: secret-scrub`) printed by `explain` when absent, in which case the class is still `sensitive` and the receipt says no scrub ran.

## Amendment 3 · 2026-11-09 · a secret has a producer (user story US-09, L9.1)

**Context.** `secret rotate` writes a new version to the vault. For a database password that does nothing: the database user still has the old one until something changes it there. The walkthrough rotated the vault only.

**Decision.** A secret declares its `producer`, a package hook that makes the new value true at the system of record (the database operator's user reconciliation, a cloud key rotation API, a certificate issuer). `secret rotate <name>` runs the hook first and writes the version only on the hook's success; on failure nothing is written and the exit is 1 with the hook's reason. A secret with no producer requires `secret rotate --manual`, which prompts for the value server-side (never echoing it) and records the operator's identity and the prompt in the receipt. The producer also declares `rotation: dual | cutover` so the session knows whether two versions may be valid during reload.

**Consequences.** Rotation becomes a transaction across the system of record and the vault, with the hook as the first phase. The annotation database gains `producer` and `rotation` on secret entries. `fuser` and `retire` are unchanged.

## Amendment 4 · 2026-11-09 · a mount declares its secret source (user story US-10, L10.6)

**Context.** References resolve against a central vault. A pod at a site whose link is down cannot resolve them and sits in creation forever, which is a stuck pod rather than an answer.

**Decision.** A mount declares where its references resolve: `secrets=central` (the default) or `secrets=site-replica`, naming a replica the platform runs at the site with a visible sync age under `/sys/<class>/<id>/secrets`. `explain` prints the source and the age for every reference a plan uses. A reference with no reachable source at plan time is exit 3 with the reference and the source named, never a pending pod. Replicas receive versions by the platform's sync, and `fuser` reports a replica that lags as a holder of the old version.

**Consequences.** Rotation at a disconnected site completes when the link returns, and the reference index knows it has not. `secret retire` of a version a replica still holds is refused like any reachable reference (amendment 1).

## Amendment 5 · 2026-11-10 · scrubbing a leaked version (user story US-09, open finding L9.7)

**Context.** Amendment 2 scans what a process writes at the moment it writes it. A value that slipped past that scan, or leaked before it existed, stays in every retained artifact, log segment and audit segment that captured it, and in every artifact computed from those; rotation (amendment 3) makes the value useless at the system of record but removes none of the copies. Artifacts are content-addressed and may be under a hold with an object lock (ADR 0006 amendments 1 and 2), so rewriting them in place would break content ids, holds and `replay --verify`. Log stores differ: some delete by query, some only whole streams, some not at all, and some forward to systems the platform does not run.

**Decision.**

1. **`secret scrub <name> <vN>` is a process.** It plans, `explain` prints its scope, the bytes to scan per store and the price, and it ends in a receipt. `secret rotate <name> --leaked` rotates (amendment 3), starts the scrub of the previous version and starts its two-phase retire (rule L9.4), under one incident id in each receipt.
2. **The scope follows the reference index and lineage.** Candidates are the artifacts, log segments and audit segments of every process the reference index (amendment 1) records as having mounted that version between its creation and now, every artifact whose inputs include a candidate, transitively, and the processes `fuser --unknown` reports. `--scope=all <path>` adds a full scan of a store or prefix, priced separately. Candidates on an unreachable device are listed, never assumed clean (ADR 0011 amendment 4). The scope also follows every write those processes made to a device (topic offset ranges, table commits, object keys, read from their receipts) and continues forward from every read of those ranges that a receipt records, transitively, so a backfill that read the topic and wrote a table is a candidate although it never mounted the secret; each candidate is listed with the path that reached it (revised 2026-11-10, L22.1).
3. **Matching never handles the value.** The vault gives the scrubber keyed hashes of the value and of its common encodings (base64, URL, JSON-escaped, hex); the scanner hashes windows of the value's length and confirms with the keyed hash, decompressing formats its driver knows. Values split across records, encrypted or transformed escape it; the receipt says `scrub: best-effort, encodings vN`, as amendment 2 does.
4. **A match is quarantined, not rewritten.** The artifact or segment moves to class `sensitive-quarantine` (amendment 2): its bytes stay identical, so content ids, holds and `replay --verify` stay valid; its readers narrow to the platform's security group; it is never served as a cache hit; and every receipt that depends on it is marked `input quarantined`, with `replay` refused to anyone outside that group. `--redact` writes a redacted copy as a new artifact marked `redacted-from <id>` for readers who need the rest.
5. **Purging needs a second person and never breaks a hold.** `secret scrub --purge` deletes the matched artifacts that no hold covers, after an approval through ADR 0017; a held match is refused for purge, naming the hold and its expiry, because the hold is a legal obligation the scrub cannot override.
6. **Logs are scrubbed as far as their store allows, and the receipt says how far.** Each log store reports `caps: log-delete` as `query` (delete matching lines, as Loki and Elasticsearch can), `stream` (narrow readers on the whole stream, purge only whole streams with `--purge`) or `none`. Destinations the platform forwards logs to are listed by `caps: log-forwards`, and the receipt names them as `outside reach`, never as scrubbed.
7. **Session stores are covered by what they already are.** Audit segments are immutable and are quarantined like held artifacts; terminal scrollback is memory-only and ends with the session (ADR 0020); a value the redactor flags never reaches the journal (ADR 0020 rule L21.2).
8. **Anyone in the secret's owning group may scrub, and nobody sees content.** The scan runs in the controller under a scrub grant across groups; results name artifact ids, offsets and owning groups, never content, and every owning group with a match is told through its page route.
9. **Exit and receipt.** Exit 0 when every candidate was scanned and every match quarantined; exit 0 with the degraded flag (ADR 0004) when a store was unreachable, a log store has `log-delete: none`, or a forward is outside reach; the receipt lists bytes scanned per store, matches, the action on each, what could not be reached and why.

**Consequences.** "Prove the old one is dead" stays the job of rotation and retire; the scrub reduces where the value can still be read and says honestly where it cannot. The artifact store needs a reverse lineage index (descendants of an artifact), which M4 carries; log stores gain two capabilities; `secret` gains `scrub` and `rotate --leaked` (ADR 0013 decision 9). L9.7 is drafted here. [US-22](../user-stories/US-22-leak-in-the-stream.md) probes it with a password leaked into an Avro topic and copied into a held lake table; it raised one revision, applied inline the same day (L22.1: the scope follows device writes and reads forward from them, decision 2), and rules L22.2, L22.4 to L22.6 and L22.8, written below, with L22.9 accepted.

## Rules · 2026-11-09 · from the user-story probes

Each line is a `rule` finding from `docs/user-stories/README.md` written here as decided; the story holds the reasoning.

- **L6.4** (US-06) `backup` excludes Secret objects by default; `--include-secrets` is audited; the receipt lists exclusions.
- **L9.2** (US-09) the producer declares `rotation: dual` (both valid for a window) or `cutover` (rotate and reload all consumers as one coordinated group in index order, with a declared blackout); `explain` prints the strategy and the blackout.
- **L9.4** (US-09) `secret retire` is two-phase: disable (version kept, reads denied and logged) for a cooling period, then delete; `fuser --unknown` lists Secret objects referenced by any pod path (env, envFrom, volumes, CSI) that the index did not create; denied reads reopen the retirement.
- **L9.8** (US-09) `secret retire --force` requires a second approval through the approvals object (US-03 L3.1); the deferred object shows its age in `ps`.
- **L17.8** (US-17) the receipt of `secret set` records the source of the value (`prompt`, `stdin` or `producer`); `secret list --actor <user>` marks prompt and stdin versions `known to actor`; a producer-set version is known to nobody, which is the reason to prefer producers (amendment 3).
- **L22.2** (US-22) each device class reports `scrub:` in `caps`: tables `row-delete` (a delete commit under `--purge`, then expiry of unheld snapshots that contain the rows), topics `none`, `truncate` or `key-tombstone` on compacted keyed topics; the scrub never truncates by default, `--purge` on a `truncate` topic lists the records it would destroy, and every match it cannot remove is reported with the date retention or expiry removes it.
- **L22.4** (US-22) for every topic range with matches the scrub lists the consumer groups whose committed offsets passed it, with their owner or `?` (L12.6), pages each owning group and lists them as `outside reach`.
- **L22.5** (US-22) the purge gate requires, for each object to be purged, an approver from that object's owning group (ADR 0016 decision 7), and the commit or deletion records the approver.
- **L22.6** (US-22) match locations in a scrub receipt are readable by the security group and by the owning group of each matched object only; the rest of the receipt follows rule L17.6.
- **L22.8** (US-22) where a schema is known the scrub scans only string and bytes fields, and the band is in `explain`; `--budget` binds, the scan runs in order of exposure (logs, artifacts, device ranges) and a degraded receipt lists the ranges left unscanned.
- **L26.6, L26.14** (US-26) artifacts written from telemetry reads keep the source's retention and are reached by its scrub; telemetry reads pass the session's redactor and report matches to the store's owning group (full rules in ADR 0023).
- **L27.2** (US-27) Terraform state and plan output are a secret sink: reads pass the scrub and withhold the attributes the provider marks sensitive, by name; the state device (ADR 0024 decision 7) is readable by the platform group, and by owning groups for their own module instances only.
- **L27.11** (US-27) `secret move <path> --to=<source>` rotates each secret on the target by default (a new value from the producer of amendment 3, the old version retired after cutover); `--copy-values` copies values controller-side between the two vaults under an approved grant (ADR 0017), never through the session; the receipt lists each secret and which way it moved.
- **L28.11** (US-28) sensitive contract inputs of a device are secret references, never values in the module instance; where the runner publishes plans is declared and warned on (full rule in ADR 0024).
- **L30.14** (US-30) the layer scrub of rule L24.9 runs on every copy of an image between registries, and copies leave no registry's residency (full rule in [ADR 0026](0026-a-registry-is-a-device-a-tag-is-a-name-for-a-digest.md)).
- **L31.13** (US-31) a message carries the scrubbed `explain` summary, never stderr or argument values (full rule in [ADR 0027](0027-convoy-tells-a-group-about-its-own-objects.md)).
- **L32.14** (US-32) shuffle files are data at rest on a device with residency and retention, owned per process and removed when it ends (full rule in [ADR 0028](0028-memory-has-a-distance-and-a-domain-is-claimed-whole.md)).
- **L34.6** (US-34) a notebook that read restricted data is scrubbed and stripped of outputs when it leaves the home device (full rule in [ADR 0030](0030-an-interactive-process-holds-a-computer-while-someone-is-attached.md)).
- **L35.5, L35.17** (US-35) secrets for an edge computer are scoped to the device and come from a site-local source; trust roots reach devices before images that need them (full rules in [ADR 0031](0031-an-edge-computer-is-reached-by-contact-and-known-by-what-it-delivers.md)).
