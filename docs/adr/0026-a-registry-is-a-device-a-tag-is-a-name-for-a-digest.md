# ADR 0026: A registry is a device, and a tag is a name for a digest

**Status:** Draft · 2026-11-13 · rules from user story US-30 added 2026-11-13
**Supersedes / relates to:** open finding L30.1 (registries are not a device class); rule L8.6 (the held registry, `/dev/registry/by-id/held`), which becomes one device of this class; ADR 0011 decisions 2, 3, 6, 7 and 9 and amendments 1 to 4 (`by-id` and views, mounts, `all`, `/sys`, the same rules for every class, drivers from packages, source repositories and artifact stores, `residency`, `disconnected`); ADR 0001 (content ids); ADR 0006 and its amendment 1 (holds, retention); ADR 0009 (receipts); ADR 0010 and its amendment 1 (packages, intent); ADR 0013 decisions 3, 6, 7 and 8 and rules L24.2 and L24.9 (`explain`, `replay`, unit verbs, device verbs, platform lists, layer scrub); ADR 0016 decisions 2, 4, 5 and 7 (rights to groups, open is a credential exchange, `svc:<group>`, approvals by a group); ADR 0017 (approvals); ADR 0018 (admission); ADR 0022 decisions 4 and 5 (`image-build`, `caps: dev-registry`, provenance); ADR 0024 (installed by Terraform); ADR 0025 decisions 2 and 4 (image formats, placement); US-30 findings L30.2 to L30.17

## Context

Every process in this design runs an image, and every record names it by digest: units, receipts, holds, builds, the platform list that rule L24.2 checks and the conversions of rule L29.11. The place those digests live has never been designed. A registry appears twice: as `/dev/registry/by-id/held`, which rule L8.6 named so that holds could copy images into it, and as `caps: dev-registry`, the development path `image-build` pushes to (ADR 0022 decision 4). There is no listing of repositories, tags or digests, no way to see a digest's signatures or SBOM, no residency, no price and no rule for what a tag means.

US-30 ran a vulnerability response across the registries a platform actually has: a production registry per residency with cross-region replication, development paths with a lifecycle policy, a pull-through cache of public registries, an edge mirror and the held registry. Every step that went wrong happened in the registries, out of the shell's sight: a base image resolved from a stale cache, a scan older than the vulnerability, a copy that dropped signatures, a tag moved behind the reconciler, a mirror without the digest, a lifecycle policy that expired images still running, pulls with a node role that could read every repository, and a replication rule that took an EU repository to the US.

Facts that constrain the design:

- The OCI distribution specification is what every registry speaks: repositories hold manifests addressed by digest, tags name manifests and can move, an index lists the manifests per platform, and since version 1.1 the referrers API lists artifacts (signatures, SBOMs, attestations, scan reports) whose `subject` is a digest. Registries without the referrers API store them under a tag schema the clients agree on.
- Blobs are shared across repositories and deduplicated by the provider, so storage is not the sum of image sizes.
- Deleting a manifest does not free bytes; the registry's garbage collection does, later, and on its own schedule. Lifecycle policies (ECR lifecycle rules, Artifact Registry cleanup policies, Harbor retention) delete by age, count and tag pattern, with no knowledge of what still runs.
- Nodes pull images, not sessions. The credential a node pulls with is the kubelet's (a node role, an image pull secret, or a credential provider per namespace where the provider supports one).
- Signatures are verified at admission (the sigstore policy controller, Kyverno, Ratify), per cluster, in enforce or audit mode.

## Decision

1. **Registries are devices under `/dev/registry/`.** The `by-id` key is the provider's own identifier (an ECR registry id and region, an Artifact Registry project, location and repository, an ACR name, a Harbor host and project), with views by provider and region, by label and by kind: `registry` (where images are pushed), `cache` (a pull-through cache of an upstream), `mirror` (a copy at a site, filled from another registry) and `held` (the registry of rule L8.6). Registries are installed by a module (ADR 0024): lifecycle policies, replication rules, immutability and pull-through upstreams are module inputs, and `/sys/registry/<id>/` shows them with the residency, the price file and, for a mirror, its source and link state (`disconnected` per ADR 0011 amendment 4).

2. **Inside a mount, a repository is a directory, a digest is a file, and a tag is a symbolic link.** A registry is mounted like any device (`mount /dev/registry/by-id/<id> /registries/prod-us`). Under the mount, repositories are directories; each manifest is a file named by its digest (`sha256:44ab…`), immutable, whose `cat` prints the manifest; each tag is a symbolic link to a digest (`stable -> sha256:44ab…`). `ls -l` on a digest prints the media type or artifact type, the platforms of an index, the size as unique and shared bytes, when and by whom it was pushed, and its provenance (ADR 0022 decision 5); `ls -l` on a tag prints the digest it resolves to and who moved it last. OCI artifacts that are not images (Helm charts, device packages and their modules, model weights, the SIF images of rule L29.11) are digests in the same tree, with their artifact type.

   ```
   /registries/prod-us/prod/orders/
   ├── sha256:44ab…          # an image index: linux/amd64, linux/arm64
   ├── sha256:44ab….ref/
   │   ├── sig/sha256:7d01…  # cosign signature by svc:ci
   │   ├── sbom/sha256:c3e9… # SPDX
   │   ├── att/sha256:a512…  # SLSA provenance
   │   └── scan/sha256:e88f… # scanner, database version, time
   ├── sha256:9f1c…
   ├── stable -> sha256:44ab…
   └── 2026.11.13 -> sha256:44ab…
   ```

3. **Referrers are files beside their subject.** `<digest>.ref/` lists the artifacts whose subject is the digest, grouped by type (`sig`, `sbom`, `att`, `scan`), read through the referrers API or the tag schema where the registry lacks it. They are read with `cat` and `ls`, never written from the session except by the executables that produce them (the builder, CI, the scanner package), each under its own identity. A scan result carries the scanner, its database version and the scan time, so its age is a fact, not an assumption.

4. **A name is resolved once, at plan time, and the digest is what everything records.** Every reference by tag (a `FROM` line, `run --image`, `release --image`, a chart, a package) is resolved to a digest at plan time; `explain` prints the tag, the digest, the registry it came through (upstream, cache or mirror) and its age; receipts, units, holds and build caches record the digest, and the tag only as "resolved from". A production profile takes no tag at all: a `release` names a digest, or a tag that `explain` resolves and the approval then pins.

5. **Verbs keep their POSIX meaning with registry semantics.**
   - `cp <digest> <path>` copies a manifest between registries by digest with all its referrers, and is how promotion works; a copy that cannot carry the referrers is refused, naming them. `cp` to a registry whose residency does not cover the source's is refused (ADR 0011 amendment 3), and the layer scrub of rule L24.9 runs on every copy.
   - `ln -sf <digest> <tag>` moves a tag. A tag that a GitOps repository, CI or a replication rule owns is written as intent to that owner (ADR 0010 amendment 1) or refused, naming it.
   - `rm <digest>` deletes the manifest, and `explain rm` says that the bytes go at the registry's next garbage collection. `rm` is refused while the digest is pinned (decision 6).
   - `du` reports unique and shared bytes apart, so the total over a registry matches the provider's storage bill. `df` reports the registry's storage and its monthly price.
   - `fuser <digest>` lists every pod and process in the mounted computers that runs it, marking those Convoy did not create. `ls --base=<digest>` lists the images built on a base, from build receipts and SBOM referrers.
   - Pushing from the session is not a verb: images reach a registry from `image-build` (ADR 0022 decision 4), from CI, or by `cp`.

6. **What is in use is pinned, and no deletion passes a pin.** A digest is pinned while a unit or a pod in a mounted computer runs it, while a receipt within its retention names it (ADR 0006), and while a hold covers it. Pins are kept by the driver in the registry's own terms: a protected tag in a platform namespace (`convoy-pin/…`) that lifecycle policies exclude, or the provider's immutability, or a copy into `held` where neither exists; `/sys/registry/<id>/pins` lists them with their reason. The inventory reports any lifecycle policy that would expire a pinned digest, and `explain` of a module change that would do so prints it.

7. **Repositories belong to groups, and pulls are as the pod's group.** A repository path is owned by a group (ADR 0016 decision 2), and a grant to another group is a change on the repository, recorded. Admission refuses a pod whose image is in a repository its group neither owns nor has been granted (ADR 0018). Where the provider allows it, nodes pull with a credential scoped to the pod's group (a kubelet credential provider per namespace, workload identity); the node's own role reads only the platform's base and system repositories. The session's own reads of a registry are as the user (ADR 0016 decision 4).

8. **Signatures are verified at admission, and the shell says whether they are.** The platform names the trusted signers per profile (for production, `svc:ci`). Each computer reports `verify=enforce|audit|none` in `caps` (ADR 0025 decision 2); `explain` and `release` print it, and a mount with `verify=required` refuses a computer that does not enforce. A release gate on vulnerabilities reads the `scan` referrers, treats a scan older than the scanner's current database or `scan_max_age` as stale, and accepts a refused digest only under an exception: an approval object (ADR 0017) from the approving group, pinned to the digest and the vulnerability, with an expiry.

9. **Where a computer pulls from is a fact, and a release checks it.** `/sys/compute/<id>/` names the registry or mirror each computer pulls from. The image is an input of the stage for locality and egress (ADR 0003 amendment 1, ADR 0025 decision 4): the pull is from the nearest replica or mirror, and `explain` prints the bytes per node and the price. `release` checks that the digest is present in the mirror the target pulls from before any replica is replaced, copies it first when the mirror is reachable, and refuses, naming the mirror, when it is `disconnected`.

10. **Public registries are upstreams, not mounts, in production.** A production profile reaches Docker Hub, `nvcr.io`, `quay.io` and other public registries only through a platform `cache` or `mirror` device, by digest; `explain` prints the upstream reference and when it was mirrored, and mirrored digests are pinned for the retention of any receipt that names them (decision 6), so an upstream deletion never breaks `replay`.

## Declined

| Proposal | Why not | What replaces it |
|---|---|---|
| registries as object stores under `/dev/obj` | blobs are content-addressed, but manifests, tags, indexes and referrers have semantics an object store does not | a class of its own (decisions 1 to 3) |
| Convoy running its own registry | a second registry to operate, back up and secure | the platform's registries as devices, installed by modules (decision 1) |
| tags as the names in receipts and units | tags move; a receipt that names a tag names nothing | digests everywhere, tags resolved once at plan time (decision 4) |
| signing from the session | every user would be a signer, and the production policy trusts CI | signatures by the builder, CI or the scanner under their identities; the session reads them (decisions 3, 8) |
| one global view merging every registry | partial failures and no single answer for a write, as ADR 0011 declined for clusters | one device per registry, `all` as the read-only union of ADR 0011 decision 6 |
| deleting by age without regard to use | lifecycle policies already do this, and US-30 lost running images to them | pins (decision 6) |

## Consequences

- L30.1 is drafted here. [US-30](../user-stories/US-30-the-base-image-has-a-cve.md) was applied on 2026-11-13: L30.6 revised ADR 0003 amendment 1 inline (the image is a placement input) and L30.7 revised ADR 0022 decision 4 inline (every `FROM` resolved to a digest and used as the cache key), which decisions 4 and 9 assume; its thirteen rules are in the Rules section below, with pointers in ADRs 0001, 0006, 0007, 0009, 0010, 0011, 0013, 0016, 0017, 0018, 0019, 0022, 0024 and 0025; L30.16 is accepted.
- Rule L8.6's `/dev/registry/by-id/held` is a device of this class with kind `held`. `caps: dev-registry` (ADR 0022) names a path in a registry device, `/registries/<id>/dev/<group>/`. ADR 0011 decision 9 gains the class.
- New paths: `/dev/registry/`, `/sys/registry/<id>/` with `pins`, `<digest>.ref/`. New `caps` keys: `registry:<kind>`, `verify=` on computers. New platform defaults: trusted signers per profile, `scan_max_age`, the pin mode per registry, the cache or mirror each profile reaches public registries through.
- ROADMAP: the registry driver, the tree and referrers are M5 with mounts; pins against receipts and holds are M4; repository ownership, scoped pulls, trusted signers and exceptions are M7; `fuser` on a digest, mirror presence and the inventory's lifecycle check are M8; registry driver packages and modules are M9; the image as a placement input is M3.
- Open, to be settled against this record: very large artifacts (model weights of hundreds of gigabytes) that are better served from object stores; export and import of digests with their referrers for sites with no link at all (an OCI layout on removable media); key management for signers that are not keyless; garbage collection that the registry runs on its own schedule and that Convoy can only observe; a story that probes this record.

## Rules · 2026-11-13 · from the user-story probes

Each line is a `rule` finding from `docs/user-stories/README.md` written here as decided; the story holds the reasoning.

- **L30.2** (US-30) a tag is a name that points to a digest; `ls -l` shows the digest and who moved it last; moving a tag that a GitOps repository, a replication rule or CI owns is written as intent to that owner or refused, naming it; on a mount with `provenance=ci`, tags are resolved only at plan time and the digest is pinned (decisions 2, 4, 5).
- **L30.3** (US-30) `rm` of a digest that a running unit, a receipt within retention or a hold pins is refused naming them; deletion is of the manifest, and `explain rm` says the bytes go at the registry's garbage collection; `du` reports unique and shared bytes apart, matching the provider's bill (decisions 5, 6).
- **L30.4** (US-30) a lifecycle policy is a fact of the registry device, set as a module input; the inventory reports digests it would expire that a unit runs, a receipt within retention names or a hold covers, and protects them before it runs, by a registry-side pin where the provider has one, else by a copy to `held` (decision 6).
- **L30.5** (US-30) repositories are owned by groups; admission refuses a pod whose image is in a repository its group neither owns nor has been granted; where the provider supports it, pulls use credentials scoped to the pod's group rather than the node's role (decision 7).
- **L30.8** (US-30) a scan result is a referrer carrying the scanner, its database version and the scan time; a release gate treats a scan older than the scanner's current database or `scan_max_age` as stale, and rescans or refuses, naming the age (decisions 3, 8).
- **L30.9** (US-30) `cp` of an image copies its referrers (signatures, SBOMs, attestations, scan reports) with it, or is refused naming the referrers it cannot copy; promotion is `cp` by digest, never a rebuild (decision 5).
- **L30.10** (US-30) each computer reports `verify=enforce|audit|none` in `caps`; `explain` and `release` print it, and a mount with `verify=required` refuses a computer that does not enforce (decision 8).
- **L30.11** (US-30) the registry or mirror a computer pulls from is in `/sys/compute/<id>/`; `release` checks the digest is present there before replacing any replica, copies it first when the mirror is reachable, and refuses naming the mirror when it is `disconnected` (decision 9).
- **L30.12** (US-30) `fuser` on a digest lists every pod in the mounted computers that runs it, marking those Convoy did not create; `ls --base=<digest>` lists images by their base digest from build receipts and SBOM referrers (decision 5).
- **L30.13** (US-30) a vulnerability exception is an approval object from an approving group, pinned to a digest and a vulnerability, with an expiry; the release gate honours it until then and refuses after, and the receipt of every release under it names it (decision 8).
- **L30.14** (US-30) a registry device and each repository carry a residency; a replication rule or `cp` to a registry outside it is refused, and the layer scrub of rule L24.9 applies to every copy (decisions 1, 5).
- **L30.15** (US-30) the receipt records the digest of every container that ran in a pod, read from the pod's status, injected sidecars and init containers included and marked so.
- **L30.17** (US-30) on a production profile public images are used only by digest from a platform cache or mirror; `explain` prints the upstream reference and when it was mirrored, and mirrored digests are pinned for the retention of any receipt that names them (decision 10).
- **L31.17** (US-31) a vulnerability exception raises `expiring` to its owning group before it lapses (full rule in [ADR 0027](0027-convoy-tells-a-group-about-its-own-objects.md)).
