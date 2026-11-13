# US-30 · The base image has a critical CVE: find every image that carries it, rebuild and promote by digest across registries and regions, mirror to the edge, and know what still runs it

| Field | Value |
|---|---|
| Primary persona | SEC |
| Secondary personas | PE, DEV, DO, SRE, FIN, ML |
| Workload class | fleet operation: a vulnerability response across the platform's OCI registries (a production registry per residency, group development paths, a pull-through cache of public registries, an edge mirror, the held registry), with rebuilds, promotion, signatures, SBOMs, a time-bound exception and lifecycle policies |
| Walkthrough | written against [ADR 0011](../adr/0011-device-tree-and-mount-table.md) decision 9 and amendments 1, 2 and 4, [ADR 0006](../adr/0006-detach-is-not-destroy.md) and rule L8.6 (the held registry), [ADR 0022](../adr/0022-the-laptop-lends-a-tree-the-cluster-builds-it.md) decisions 4 and 5, [ADR 0013](../adr/0013-native-verbs-have-one-shape.md) decision 7 and rules L24.2 and L24.9, [ADR 0003](../adr/0003-three-backends-local-first.md) amendment 1 as revised by L29.7, [ADR 0009](../adr/0009-stdout-artifact-stderr-log.md), [ADR 0016](../adr/0016-identity-is-a-user-rights-belong-to-groups.md), [ADR 0017](../adr/0017-approvals-are-objects-on-the-process.md), [ADR 0018](../adr/0018-labels-are-the-signal-admission-is-the-decision.md), [ADR 0007](../adr/0007-secrets-are-references.md) amendment 2, [ADR 0024](../adr/0024-a-device-is-installed-by-terraform.md) and [ADR 0025](../adr/0025-a-computer-is-a-device-kubernetes-is-one-driver.md); it follows US-29 |
| Gaps already closed there | none |
| Milestones needed | M3, M4, M5, M7, M8, M9, M11 |
| Findings in this document | L30.1 – L30.17 |

## 1. Story

**As a** security engineer, **I want** to learn within an hour which images carry a critical vulnerability, which of them are running and where, have them rebuilt and promoted by digest to every registry and site that serves production, and grant a dated exception for the one service that cannot move today, **so that** the fix reaches every region and the edge without anyone retagging by hand, nothing unsigned or unscanned slips through on the way, and the auditor can see afterwards exactly what ran, when, and under which exception.

**Context.** After US-29 the platform runs ADR 0025. Its registries, each installed by a module (ADR 0024):
- **US production:** an ECR registry in us-east-1, which CI pushes to and where a replication rule copies `prod/*` to us-west-2.
- **EU production:** an Artifact Registry in europe-west4 for the EU-resident groups.
- **Development:** the groups' development paths `dev/<group>/` in ECR (`caps: dev-registry`, ADR 0022 decision 4), with a lifecycle policy that expires untagged digests after 14 days.
- **Public images:** an ECR pull-through cache of Docker Hub and `nvcr.io`.
- **Edge:** a Harbor mirror at the edge site `sp-01`, which pulls from ECR over the site's link (US-10).
- **Holds:** the held registry of rule L8.6.

Admission on the production clusters verifies cosign signatures with the sigstore policy controller. `acme/base:2026.10`, built from `debian:bookworm-slim`, is the base of 41 service images. On Tuesday at 08:10 a critical CVE in glibc, CVE-2026-48211, is published. `amara` (user `amara`, group `security`) leads the response.

## 2. Actors and identities

| Role | Who | Identity Convoy sees | Notes |
|---|---|---|---|
| Leads the response | `amara` | user `amara`, group `security` | |
| Own the services | 14 groups | `svc:<group>` for builds and releases | |
| Builds and signs | CI | `svc:ci` | the only signer the production policy trusts |
| Grants exceptions | `security-leads` | members | an ADR 0017 approving group |
| Operates the registries | platform team | `svc:platform` | modules, replication rules, lifecycle policies |
| Operates the edge site | `site-ops` | group `site-ops` | the link is down for hours at a time |
| Pays | each group | storage and egress | |

## 3. Preconditions

- **Capabilities** (`caps`): `dev-registry` on ECR. On the production clusters, `admission:sigstore`. A scanner package provides `scan:trivy`. On `sp-01`, `mirror:harbor`.
- **Packages**: `registry` with `aws-ecr`, `gcp-artifact-registry` and `harbor` modules; the sigstore policy controller; a scanner that attaches SBOMs and scan reports as OCI referrers.
- **Platform defaults** `explain` must print: the registry each mount pulls from, the signature policy and whether it is enforced, the provenance level (ADR 0022 decision 5), the lifecycle policies of each registry, and the residency of each registry.

## 4. Main path

```bash
# Tuesday 08:10, CVE-2026-48211 in glibc 2.36
[amara] $ ls /dev/registry
ls: /dev/registry: no such device class                                                         (L30.1)
#  registries appear as /dev/registry/by-id/held (L8.6) and as caps: dev-registry, nothing else;
#  which images carry glibc 2.36? the SBOMs exist as referrers in ECR; the shell cannot list them

# 08:40 the platform team lists them with a script against three registry APIs
#  41 repositories FROM acme/base; 6 more FROM debian directly; 3 vendor images through the
#  pull-through cache; the inference server's model image on nvcr.io
[amara] $ fuser /dev/registry/…/acme/orders@sha256:9f1c…
#  no answer: fuser knows devices, not images; and the Istio sidecar injected into every pod
#  is FROM debian too, and no receipt names it                                                  (L30.12, L30.15)

# 09:30 rebuild the base
[amara] $ run image-build /src/base --file=Dockerfile
#  FROM debian:bookworm-slim resolved through the pull-through cache to the digest cached on
#  Sunday, before Debian's fix; the build "succeeds" with the old glibc                         (L30.7, L30.17)
#  the scan report referrer on yesterday's base still says "0 critical": the scan predates
#  the CVE                                                                                        (L30.8)

# 11:00 rebuild 41 services; CI signs; promotion
[ci] $ crane copy …/prod/orders@sha256:44ab… europe-west4-docker.pkg.dev/…/orders
#  the copy carries the manifest, not its signature or SBOM; prod-eu's admission is in audit
#  mode since a migration in August and admits it unsigned                                       (L30.9, L30.10)
[dev] $ crane tag …/prod/orders:stable sha256:44ab…
#  stable is the tag the GitOps repository of one team still names; the reconciler rolls it
#  out with no release, no receipt, no approval                                                   (L30.2)

# 13:00 the edge site
[sre] $ release /clusters/sp-01/orders --image=sha256:44ab…
#  the rollout starts; Harbor has not mirrored the digest; the link is down; pods sit in
#  ImagePullBackOff and the old replicas are already scaled to two                              (L30.11)

# 14:00 an exception
#  payments-legacy cannot rebuild today (a vendor library pins glibc); security-leads agree in
#  chat to accept it until Friday; nothing records it, nothing expires it                        (L30.13)

# 15:30 a GPU stage of US-29 is placed in europe-west4 and pulls an 11 GB image from us-east-1
#  on each of 32 nodes                                                                            (L30.6)

# Wednesday: cleanup
[dev] $ rm …/dev/vision/train@sha256:0bd2…
#  deleted; the receipt of last week's training run names it, and replay --verify fails           (L30.3)
#  the lifecycle policy expired three untagged digests in dev/ that two branch units still run;
#  a node drained overnight, and the pods cannot pull                                              (L30.4)
[fin] $ du -sh /dev/registry/…/prod
#  each repository counts the shared base layers again; the total is three times the bill          (L30.3)

# Thursday: the auditor's questions
#  which digests ran in prod-eu between Tuesday 08:10 and Thursday, signed by whom?
#  a group's unit ran another group's private image by digest: nodes pull with the node role,
#  which reads every repository                                                                    (L30.5)
#  the EU repository of the clinical group is replicated to us-east-1 by a rule added for DR       (L30.14)
#  two scanners disagree on whether a vendor image is affected                                     (L30.16)
```

| Step | Convoy object | Registry or cluster object | Where the state lives |
|---|---|---|---|
| finding affected images | none today | SBOM and scan referrers | each registry |
| `image-build` | a build process with a receipt | a manifest pushed to a development path | the registry, the artifact store (cache) |
| promotion | none today (`crane copy` beside the shell) | manifests and referrers copied between registries | two registries |
| `release` | a unit release as intent | a Deployment's image digest | the GitOps repository |
| an exception | none today | none | chat |

## 5. Abstraction coverage

| # | Aspect | Touched | How this story uses it | What to verify |
|---|---|---|---|---|
| A1 | Pipe: staged artifact, content id, cache | ● | a build cache keyed by a tag (L30.7); digests as content ids across registries (L30.9) | AC-30.7, AC-30.9 |
| A2 | Command: class, annotation, executable fields | ○ | `image-build` and its base | |
| A3 | Session: mount table, env, cwd, attach and detach | ○ | registry mounts | |
| A4 | Backend selection | ● | placement ignores where the image is (L30.6) | AC-30.6 |
| A5 | Completion contract | ○ | a release that cannot pull | |
| A6 | Device tree | ● | no registry class (L30.1); `fuser` on a digest (L30.12) | AC-30.12 |
| A7 | Signals and lifecycle | – | | |
| A8 | Secrets as references | ● | pull credentials; layer scrub on copy (L30.5, L30.14) | AC-30.5 |
| A9 | Cost | ● | shared layers, replicas, cross-region pulls (L30.3, L30.6) | AC-30.3 |
| A10 | Identity and authorization | ● | pulls by node role (L30.5); signers; exceptions (L30.13) | AC-30.5, AC-30.13 |
| A11 | Platform defaults and `caps` | ● | signature enforcement per cluster (L30.10); lifecycle policies (L30.4) | AC-30.10 |
| A12 | Storage | ● | `rm` and garbage collection of digests, `du` of layers (L30.3) | AC-30.3 |
| A13 | Networking | ○ | pulls across regions and over an edge link | |
| A14 | Time and queueing | ○ | scan freshness (L30.8); an exception's expiry | AC-30.8 |
| A15 | Audit, receipts, retention and `replay` | ● | what ran, including injected containers (L30.15); deleted digests (L30.3) | AC-30.15 |
| A16 | Durability | ● | lifecycle policies and upstream deletions (L30.4, L30.17) | AC-30.4, AC-30.17 |

## 6. Loophole probes

| # | Probe | Question for this story | Design answer | Finding |
|---|---|---|---|---|
| P1 | Imperative verb, declarative owner | moving a tag that a GitOps repository names | ADR 0010 amendment 1 writes digests as intent; tags are not mentioned | **L30.2** |
| P2 | POSIX semantics that cannot hold | `rm` of a digest others pin; `du` of shared layers | ADR 0006 guards devices and artifacts, not manifests | **L30.3** |
| P3 | Orphaned or lost state | a lifecycle policy expires digests that units, receipts and holds use | L8.6 copies held images; nothing protects running ones | **L30.4** |
| P4 | Identity leak or escalation | nodes pull with a role that reads every repository | ADR 0016 binds rights to groups; pulls are not opens | **L30.5** |
| P5 | Invisible cost | an image in another region pulled by every node | placement reads input devices (ADR 0003 amendment 1); an image is not one | **L30.6** |
| P6 | Lying cache | (a) a base resolved from a tag through a cache; (b) a scan that predates the CVE | (a) ADR 0022 decision 4 builds from a snapshot, the base unpinned; (b) none | **L30.7**, **L30.8** |
| P7 | Silent failure | (a) a copy that drops signatures; (b) admission in audit mode | (a) none; (b) ADR 0019 decision 6 reports unenforced reach, not unenforced signatures | **L30.9**, **L30.10** |
| P8 | Partial fleet | a release to a site whose mirror lacks the digest | ADR 0011 amendment 4 makes the site `disconnected`, not the mirror | **L30.11** |
| P9 | Created elsewhere | images running in pods Convoy did not create | `fuser` lists holders of devices | **L30.12** |
| P10 | Multi-party action | an exception granted by one group for another's service | ADR 0017 approves plans, not standing exceptions | **L30.13** |
| P11 | Data at rest | a replication rule copies an EU repository to the US | residency binds mounts (ADR 0011 amendment 3), not registry rules | **L30.14** |
| P12 | Process-tree inheritance | sidecars and init containers injected by admission | receipts record the images Convoy wrote | **L30.15** |

Two questions without a probe: is a registry a device at all, and what happens to public images that change or disappear upstream? **L30.1**, open; **L30.17**, rule. Scanners that disagree are **L30.16**, accepted.

## 7. Failure matrix

| Failure | Detected by | Exit | Receipt says | Recovery verb |
|---|---|---|---|---|
| base resolved to a stale digest | build (L30.7) | 0, with the digest printed | the resolved base digest and its age | `image-build --refresh-base` |
| scan older than the vulnerability database | release gate (L30.8) | 3 | the scan's database version and date | rescan |
| copy without referrers | `cp` (L30.9) | 3 | the referrers missing | `cp --with-referrers` |
| unenforced signature policy | plan (L30.10) | 0 degraded, 3 under `verify=required` | the cluster's mode | fix admission |
| digest absent from the site's mirror | plan (L30.11) | 3, or a pre-copy | the mirror and its link | `cp` to the mirror, then `release` |
| `rm` of a pinned digest | `rm` (L30.3) | 3 | the units, receipts and holds that pin it | none, or `--force` with approval |
| expiry of a pinned digest by a lifecycle policy | inventory (L30.4) | none | the policy and the pins | pin or copy to `held` |
| exception expired | release gate (L30.13) | 3 | the exception and its expiry | rebuild, or a new exception |

## 8. Acceptance criteria

- **AC-30.2** Given a tag that a GitOps repository or a replication rule names, then moving it from the shell is refused or written as intent to its owner, naming the owner; `ls -l` on a tag shows the digest it resolves to and who moved it last.
- **AC-30.3** Given `rm` of a digest that a running unit, a receipt within retention or a hold pins, then it is refused naming them; `du` on a registry path reports unique and shared bytes apart, and the total matches the provider's storage bill.
- **AC-30.4** Given a lifecycle policy that would expire a digest a unit runs, a receipt within retention names, or a hold covers, then `explain` and the inventory report it, and the digest is protected (pinned in the registry, or copied to `held`) before the policy runs.
- **AC-30.5** Given a pod of group A that names an image in a repository owned by group B, then admission refuses it unless B has granted A read, and nodes pull with credentials scoped to the pod's group.
- **AC-30.6** Given a stage placed on a computer in another region from its image, then the image counts as an input for locality and egress, the pull is from the nearest replica, and `explain` prints the pull's bytes and price.
- **AC-30.7** Given `image-build`, then every `FROM` is resolved to a digest at plan time, printed by `explain` with its age, recorded in the receipt and used as the cache key; `--refresh-base` resolves again past any cache.
- **AC-30.8** Given a release gate that reads scan results, then a scan older than the scanner's current database or `scan_max_age` is stale, and the gate rescans or refuses, naming the age.
- **AC-30.9** Given `cp` of an image between registries, then its referrers (signatures, SBOMs, attestations) are copied with it, or the copy is refused naming what is missing.
- **AC-30.10** Given a cluster's admission, then `caps` reports `verify=enforce|audit|none`; `explain` and `release` print it, and a mount with `verify=required` refuses a cluster that does not enforce.
- **AC-30.11** Given a release to a cluster that pulls through a mirror, then the plan checks the digest is in the mirror; if not, it copies the digest first, or refuses when the mirror is `disconnected`, before any replica is replaced.
- **AC-30.12** Given a digest, then `fuser` lists every pod in the mounted clusters that runs it, whether Convoy created it or not, and images can be listed by their base digest from build receipts and SBOMs.
- **AC-30.13** Given a release of a digest the vulnerability policy refuses, then an exception is an approval object pinned to the digest and the vulnerability, with an expiry; the gate honours it until the expiry and refuses after it.
- **AC-30.14** Given a registry or repository with a residency, then a replication rule or `cp` to a registry outside it is refused, and the layer scrub of rule L24.9 applies to every copy.
- **AC-30.15** Given a pod, then the receipt records the digest of every container that ran, injected sidecars and init containers included, read from the pod's status.
- **AC-30.17** Given a production profile, then public images are used only by digest from a platform mirror, and `explain` prints the upstream reference and when it was mirrored.

## 9. Persona lenses

- **SEC**: wants to go from a CVE to the list of running digests in minutes, to see that every promoted digest kept its signature and SBOM, and to grant exceptions that expire on their own.
- **PE**: wants registries to be devices with residency, lifecycle and replication visible in `explain`, and no pull that bypasses group ownership.
- **DEV**: wants a rebuild to pick up the fixed base without guessing which cache it came through, and never to lose an image a branch unit still runs.
- **DO**: wants promotion to be a copy by digest with its referrers, and tags to stop moving releases behind the reconciler's back.
- **SRE**: wants an edge release never to start before the site can pull it.
- **FIN**: wants registry storage counted once per layer and cross-region pulls priced before they happen.
- **ML**: wants an 11 GB training image pulled from the nearest replica, not across an ocean on every node.

## 10. Findings

| Id | Probe | Finding | Proposed disposition | Touches |
|---|---|---|---|---|
| L30.1 | none | Registries are not a device class. Images are referenced by digest everywhere (units, receipts, holds, builds, the platform list of rule L24.2), and a registry appears only as `/dev/registry/by-id/held` (rule L8.6) and `caps: dev-registry`; there is no `by-id`, no mount, no listing of repositories, tags, digests or referrers, no residency, no price. | **open**: a registry device class: repositories as directories, tags as names that point to digests, digests as content ids, referrers (signatures, SBOMs, attestations, scan reports) as files beside them; a driver per registry kind (ECR, Artifact Registry, ACR, Harbor, Quay, a pull-through cache, a mirror); residency, lifecycle and replication as device facts; copy, tag, delete and garbage collection with device semantics | ADR 0011 d9, a1, a2; ADR 0006; ADR 0022 d4; ADR 0013 d8 |
| L30.2 | P1 | Moving a tag that a GitOps repository names releases a new image behind the reconciler, with no release, receipt or approval. | **rule** (ADR 0010 rules; ADR 0013 rules): a tag is a name that points to a digest; `ls -l` shows the digest and who moved it last; moving a tag that a GitOps repository, a replication rule or CI owns is written as intent to that owner or refused, naming it; on a mount with `provenance=ci`, tags are resolved only at plan time and the digest is pinned | ADR 0010 a1; ADR 0013 d7; ADR 0022 d5 |
| L30.3 | P2 | `rm` of a digest succeeded although a receipt names it, so `replay --verify` fails; `du` counts shared layers once per repository and overstates storage threefold. | **rule** (ADR 0006 rules): `rm` of a digest that a running unit, a receipt within retention or a hold pins is refused naming them; deletion is of the manifest, and the bytes go at the registry's garbage collection, which `explain rm` says; `du` reports unique and shared bytes apart, matching the provider's bill | ADR 0006; ADR 0009; ADR 0013 d6 |
| L30.4 | P3 | A registry lifecycle policy expired digests that branch units still ran; a drained node could not pull them back. | **rule** (ADR 0006 rules; ADR 0024 rules): a lifecycle policy is a fact of the registry device, set as a module input; the inventory reports digests it would expire that a unit runs, a receipt within retention names or a hold covers, and protects them before it runs (a registry-side pin where the provider has one, else a copy to `held`) | ADR 0006; L8.6; ADR 0024 d6 |
| L30.5 | P4 | Nodes pull with a role that reads every repository, so a group's unit can run another group's private image by digest. | **rule** (ADR 0016 rules; ADR 0018 rules): repositories are owned by groups; admission refuses a pod whose image is in a repository its group neither owns nor has been granted (`chmod`-style grant on the repository); where the provider supports it, pulls use credentials scoped to the pod's group (the kubelet credential provider per namespace) rather than the node's role | ADR 0016 d2, d5; ADR 0018 |
| L30.6 | P5 | Placement reads the location of input devices but not of the image, so a GPU stage pulled 11 GB across regions on each of 32 nodes. | **amend** ADR 0003 amendment 1: the stage's image is one of its inputs for locality and egress; the pull is from the nearest replica the registry device lists; `explain` prints the pull's bytes per node and its price, and a missing replica in the target region is a candidate's cost, not a refusal | ADR 0003 a1; ADR 0025 d4; ADR 0010 |
| L30.7 | P6 | `image-build` resolved `FROM debian:bookworm-slim` through the pull-through cache to Sunday's digest, before Debian's fix, and the build cache was keyed by the tag. | **amend** ADR 0022 decision 4: every `FROM` is resolved to a digest at plan time, printed by `explain` with its age and source (upstream or cache), recorded in the build's receipt and used as the cache key; `--refresh-base` resolves again from upstream past any cache | ADR 0022 d4; ADR 0001; ADR 0009 |
| L30.8 | P6 | The scan report on yesterday's base said "0 critical" because the scan predated the CVE. | **rule** (ADR 0013 rules; ADR 0017 rules): a scan result is a referrer carrying the scanner, its database version and the scan time; a release gate treats a scan older than the scanner's current database or the platform's `scan_max_age` as stale, and rescans or refuses, naming the age | ADR 0013 d3; ADR 0017 |
| L30.9 | P7 | A copy between registries carried the manifest without its signature and SBOM. | **rule** (ADR 0001 rules; ADR 0013 rules): `cp` of an image copies its referrers (signatures, SBOMs, attestations, scan reports) with it, or is refused naming the referrers it cannot copy; promotion is `cp` by digest, never a rebuild | ADR 0001; ADR 0013 d8 |
| L30.10 | P7 | The production policy verifies signatures, but one cluster's admission had been in audit mode since August and admitted an unsigned digest. | **rule** (ADR 0018 rules; ADR 0019 rules): `caps` reports `verify=enforce\|audit\|none` per cluster, as ADR 0019 decision 6 reports `netpol: none`; `explain` and `release` print it, and a mount with `verify=required` refuses a cluster that does not enforce | ADR 0018 d6; ADR 0019 d6 |
| L30.11 | P8 | A release to an edge site started before its mirror held the digest, and the link was down; old replicas were scaled down while new ones could not pull. | **rule** (ADR 0011 rules; ADR 0013 rules): the registry or mirror a cluster pulls from is in `/sys/compute/<id>/`; `release` checks the digest is present there before replacing any replica, copies it first when the mirror is reachable, and refuses naming the mirror when it is `disconnected` | ADR 0011 a4; ADR 0013 d7; ADR 0025 |
| L30.12 | P9 | `fuser` cannot answer which pods run a digest, and images cannot be listed by their base, so the platform team scripted three registry APIs. | **rule** (ADR 0011 rules): `fuser` on a digest lists every pod in the mounted clusters that runs it, marking those Convoy did not create; a registry path lists images by their base digest from build receipts and SBOM referrers | ADR 0011 d7; L9.4 |
| L30.13 | P10 | An exception for a service that cannot rebuild was agreed in chat, recorded nowhere and expired never. | **rule** (ADR 0017 rules): a vulnerability exception is an approval object from an approving group, pinned to a digest and a vulnerability, with an expiry; the release gate honours it until then and refuses after, and the receipt of every release under it names it | ADR 0017 d1; ADR 0016 d7 |
| L30.14 | P11 | A replication rule added for disaster recovery copies an EU repository to us-east-1. | **rule** (ADR 0011 rules; ADR 0007 rules): a registry device and each repository carry a residency; a replication rule or `cp` to a registry outside it is refused, and the layer scrub of rule L24.9 applies to every copy | ADR 0011 a3; ADR 0007 a2; L24.9 |
| L30.15 | P12 | Admission injects sidecars and init containers (a mesh proxy, a secrets agent) whose images no receipt names, so a CVE in them is never traced to the units that ran them. | **rule** (ADR 0009 rules): the receipt records the digest of every container that ran in the pod, read from the pod's status, injected containers included and marked so | ADR 0009; ADR 0018 |
| L30.16 | none | Two scanners disagree on whether a vendor image is affected. | **accepted**: a clean scan is not proof; the gate names the scanner it trusts per profile, and the receipt records which scanner and database the decision rests on | ADR 0017 |
| L30.17 | none | A public image resolved through the pull-through cache was stale, and a public tag can be moved or deleted upstream, breaking `replay`. | **rule** (ADR 0022 rules; ADR 0010 rules): on a production profile public images are used only by digest from a platform mirror; `explain` prints the upstream reference and when it was mirrored, and mirrored digests are kept for the retention of any receipt that names them | ADR 0022 d5; ADR 0010; ADR 0006 |
