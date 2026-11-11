# US-24 · But it runs on my machine: take a service from a green laptop to test and production, and learn every difference before it bites

| Field | Value |
|---|---|
| Primary persona | DEV |
| Secondary personas | DO, SRE, PE, SEC, FIN |
| Workload class | the inner loop and promotion of one service: tests on a laptop, a working tree in the session, a unit in a test namespace from a locally built image, a release to production through CI and an approval |
| Walkthrough | written against [ADR 0005](../adr/0005-session-is-a-server-side-object.md) (the laptop is a terminal; launched processes inherit the session), [ADR 0003](../adr/0003-three-backends-local-first.md) (local means the session), [ADR 0011](../adr/0011-device-tree-and-mount-table.md) amendment 2 (source repositories are devices), [ADR 0013](../adr/0013-native-verbs-have-one-shape.md) decisions 3, 4 and 7 (`explain`, `run --unit`, `release`), [ADR 0019](../adr/0019-a-unit-has-an-address-reach-is-a-grant.md) decisions 4 and 7 (egress, `attach -L`), [ADR 0007](../adr/0007-secrets-are-references.md) (references), [ADR 0010](../adr/0010-platform-defaults-and-two-package-tiers.md) (profiles, intent) and [ADR 0017](../adr/0017-approvals-are-objects-on-the-process.md) (the release gate); it reuses the release path of US-03 |
| Gaps already closed there | none |
| Milestones needed | M2, M3, M8, M11 |
| Findings in this document | L24.1 – L24.11 |

## 1. Story

**As an** application developer, **I want** the service that passes its tests on my laptop to run the same way in the test namespace and in production, and when it cannot, to be told what is different before I release rather than by a crash afterwards, **so that** "it runs on my machine" stops being the start of an incident and I never have to learn the platform's rules one failure at a time.

**Context.** `kai` writes `orders-api`, a Python 3.12 service, on an arm64 laptop. Their loop is `docker compose up` (the service and `postgres:latest`, which is 17 today), `pytest`, and a `.env` file with a Stripe test key and `DATABASE_URL=postgres://localhost:5432/orders`. The team's environments are mounts: `/clusters/test/orders` (profile `test`: Pod Security `baseline`, `egress=declared`, `provenance=any`, a LimitRange defaulting containers to 512 Mi, amd64 nodes) and `/clusters/prod-eu/orders` and `/clusters/prod-us/orders` (profile `prod`: Pod Security `restricted` with a read-only root filesystem, `egress=declared`, `provenance=ci`, releases as intent through the GitOps repository, an approval from `orders-leads`). The test database is a cnpg cluster on PostgreSQL 16, created by the platform's operator. CI builds and signs images from the default branch and from pull requests.

## 2. Actors and identities

| Role | Who | Identity Convoy sees | Notes |
|---|---|---|---|
| Runs it | `kai` | user `kai`, group `orders` | attaches from the laptop |
| Approves production | `orders-leads` | member of `orders-leads` | the `release` gate (ADR 0017) |
| Builds images | CI | `svc:orders` through the CI's trust policy | signs and attests the digests it builds |
| Owns the test database | `orders` | group; created by the cnpg operator | version in `/sys` |
| Pays | `orders` | units in test, scans, releases | |
| Operates the platform | platform team | profiles `test` and `prod`, the LimitRange, Pod Security labels, the egress proxy, the registry, cosign verification at admission | |

## 3. Preconditions

- **Capabilities**: `scm:github`, `registry:ghcr` with `signature: cosign`, `fqdn-policy` through the egress proxy, `pod-security` per namespace, `flows` (dropped connections counted per pod).
- **Mounts**: the three cluster mounts above; `/dev/scm/by-id/github-acme/orders-api` available to the `orders` group.
- **Annotations**: none for `orders-api`; it is the team's own image.
- **Platform defaults** `explain` must print: the profile of each mount, the image a stage runs in, the image's platforms against the target nodes, the env a unit will have and where each value comes from, the egress the unit is allowed, the namespace's defaulted requests and limits and its Pod Security level, the provenance a mount requires.

## 4. Main path

```bash
# Monday, laptop: green
laptop$ docker compose up -d && pytest -q            # 212 passed
laptop$ docker build -t ghcr.io/acme/orders-api:dev . && docker push ghcr.io/acme/orders-api:dev

laptop$ convoy attach
[kai] $ cd ~/src/orders-api
cd: no such file or directory; the laptop's files are not in the session                       (L24.1)
#  kai pushes a work-in-progress branch to reach the code from the session
[kai] $ mount /dev/scm/by-id/github-acme/orders-api /src/orders-api -o branch=kai/refunds
[kai] $ cd /src/orders-api && make test
3 failed, 209 passed: SyntaxError in orders/tz.py (a 3.12 f-string)
#  the stage ran in the session's toolbox, Python 3.11, not in the image the service runs in    (L24.5)

[kai] $ export DATABASE_URL=postgres://localhost:5432/orders      # habit from the laptop
[kai] $ cd /clusters/test/orders
[kai] $ explain run --unit orders-api --image=ghcr.io/acme/orders-api:dev
unit orders-api: 1 replica, image sha256:7be1… (tag dev), egress: declared (none), runs as svc:orders
[kai] $ run --unit orders-api --image=ghcr.io/acme/orders-api:dev
unit orders-api: 0/1 ready, CrashLoopBackOff: exec format error
#  the digest is linux/arm64 only and every node in test is amd64; explain did not check     (L24.2)

laptop$ docker buildx build --platform linux/amd64,linux/arm64 -t ghcr.io/acme/orders-api:dev --push .
[kai] $ release orders-api sha256:9c4d…                   # the multi-arch digest
unit orders-api: 0/1 ready: connection refused to localhost:5432
#  the unit inherited DATABASE_URL from kai's session (ADR 0005); a release would carry it on   (L24.3)
[kai] $ unset DATABASE_URL
[kai] $ run --unit orders-api --image=sha256:9c4d… --env DATABASE_URL=ref:/secrets/orders/test-db-uri --connect=/db/orders-pg   # --env as L24.3 proposes
migration 0042 failed: function json_table does not exist
#  the laptop runs PostgreSQL 17, the test cluster 16; --connect says nothing about versions     (L24.6)

[kai] $ curl -s orders-api/refunds -d @refund.json        # through the session's hosts view
504 after 30 s: the call to api.stripe.com timed out
#  the image has no egress annotation, so egress=declared allows nothing; nothing said so        (L24.10)
[kai] $ run --unit orders-api … --egress=api.stripe.com:443        # --egress as L24.10 proposes
[kai] $ run 'k6 run load.js' --connect=/units/orders-api
orders-api-0 OOMKilled at 512 Mi (limit defaulted by LimitRange test-defaults)                   (L24.7)
#  the laptop had no limit; the namespace's default was never in explain

# Wednesday: test is green, kai releases the same digest to production
[kai] $ release orders-api sha256:9c4d… --to=/clusters/prod-eu/orders
refused (exit 3): mount prod-eu requires provenance=ci; sha256:9c4d… was pushed from a laptop
  and its layers hold a file named .env (COPY . .)                                                (L24.9)
  alternative: release the digest CI built from commit 3e1a… (sha256:e1f0…)
[kai] $ release orders-api sha256:e1f0… --to=/clusters/prod-eu/orders
gate: release to prod-eu needs an approver from orders-leads (policy prod-release)
[orders lead] $ approve 9905
intent committed to gitops/orders@5512; reconciled; orders-api 0/3 ready:
  OSError: [Errno 30] Read-only file system: '/app/.cache'
#  test is Pod Security baseline, prod is restricted with a read-only root; nobody saw the diff (L24.4)

# Two weeks later
[kai] $ ps | grep orders-api-
orders-api-refunds  orders-api-tz  orders-api-pg17  orders-api-kai2 …  6 units from branches, 14 days, $212   (L24.8)
```

| Step | Convoy object | Kubernetes or cloud object | Where the state lives |
|---|---|---|---|
| the code in the session | a mount of a source-repository device at a branch | a clone in the session's volume | the repository, the session journal |
| `make test` | a local stage | the session pod's toolbox | receipts |
| a unit in test | `run --unit` | Deployment, Service, NetworkPolicy, FQDN policy | the unit record, the namespace |
| release to production | intent and an approval | a commit to the GitOps repository, Argo CD sync, cosign check at admission | the repository, `/sys/proc/<pid>/approval/<n>` |

## 5. Abstraction coverage

| # | Aspect | Touched | How this story uses it | What to verify |
|---|---|---|---|---|
| A1 | Pipe | ○ | test results as artifacts | |
| A2 | Command | ● | the team's own image has no annotation: egress and runtime unknown (L24.10) | AC-24.6 |
| A3 | Session | ● | the laptop is a terminal; the session's env reaches a unit (L24.1, L24.3) | AC-24.2 |
| A4 | Backend | ● | a local stage runs in the toolbox, not in the service's image (L24.5) | AC-24.4 |
| A5 | Contract | ○ | the unit's readiness as the exit of `run --unit` | |
| A6 | Device tree | ● | the source repository as a device; node platforms and database versions from `/sys` (L24.2, L24.6) | AC-24.1, AC-24.5 |
| A7 | Signals | – | | |
| A8 | Secrets | ○ | references per environment; a `.env` baked into a layer (L24.9) | AC-24.8 |
| A9 | Cost | ● | branch units left running (L24.8) | AC-24.9 |
| A10 | Identity | ○ | CI's identity on a digest; the release approver | |
| A11 | Defaults | ● | two profiles that differ in six ways nobody listed (L24.4, L24.7) | AC-24.3, AC-24.7 |
| A12 | Storage | ○ | a read-only root filesystem in production | |
| A13 | Networking | ● | egress for an image with no annotation; the hosts view from the session (L24.10) | AC-24.6 |
| A14 | Time | ○ | the session's `TZ` against the unit's UTC (L24.3); branch unit lifetime | |
| A15 | Audit | ○ | the release receipt and its gate | |
| A16 | Durability | – | | |

## 6. Loophole probes

| # | Probe | Question for this story | Design answer | Finding |
|---|---|---|---|---|
| P1 | Imperative vs declarative | `release` to the GitOps-managed production namespace | written as intent, reconciled (ADR 0010 amendment 1) | ok |
| P2 | POSIX cannot hold | `make test` typed in the source tree runs with the session's tools, not the project's | the toolbox is the only image a local stage knows | **L24.5** |
| P3 | Orphaned state | units started from branches outlive the branches | nothing ends a unit but `kill` | **L24.8** |
| P4 | Identity leak | a laptop build copied `.env` into an image layer, and the digest was offered to production | a digest is a digest; nothing in the design asks who built it or what it holds | **L24.9** |
| P5 | Invisible cost | six branch units at $15 a day each | visible in `df` and `cost`, but nobody is told | folded into **L24.8** |
| P6 | Lying cache | "green in test" is a result computed from an environment that differs from production's; the release reuses it as if it held | `explain` prints one mount's plan at a time; no verb compares two | **L24.4** |
| P7 | Silent failure | (a) an arm64-only digest scheduled on amd64 nodes; (b) calls to an undeclared host dropped until a timeout | (a) `explain` checks the digest resolves, not that it runs on the target's nodes; (b) undeclared egress is "none" with no notice | **L24.2**, **L24.10** |
| P8 | Partial fleet | `prod-eu` and `prod-us` run different Kubernetes minor versions and different `caps` | each mount's `explain` covers its own; the comparison of L24.4 covers both | folded into L24.4 |
| P9 | Created elsewhere | (a) the LimitRange and Pod Security labels the platform set on the namespace; (b) the test database the operator created, and its version | (a) neither appears in `explain`; (b) `--connect` checks reach, not version | **L24.7**, **L24.6** |
| P10 | Multi-party | the production release needs CI's build and a lead's approval | provenance is checked at admission (cosign) and the gate is ADR 0017's; L24.9 moves the refusal to plan time | ok |
| P11 | Data at rest | the working tree exists only on the laptop; reaching it from the session means pushing unfinished code to a shared repository, or a path the design does not have | the laptop is a terminal (ADR 0005), and a terminal is not a device | **L24.1** |
| P12 | Inheritance | a unit launched from a session inherits the session's env, Plan 9 style, including `DATABASE_URL=localhost` and `TZ=America/Sao_Paulo` | ADR 0005 makes inheritance the rule for every launched process; a unit outlives the session and is re-created by every release | **L24.3** |

A thirteenth question: the bug in production is a race under real traffic. `kai` can read production's logs and reach its replicas with `attach -L` (ADR 0019 decision 7), but no debugger attaches to a production process, and nothing in the cluster can call back into the laptop where the IDE runs. **L24.11**, accepted.

## 7. Failure matrix

| Failure | Detected by | Exit | Receipt says | Recovery verb |
|---|---|---|---|---|
| digest without the target nodes' platform | plan (L24.2) | 3 | the digest's platforms and the pool's | rebuild for the platform |
| unit env from the session | plan (L24.3) | 3 | the variables and their source | `--env` on the unit |
| local stage in the wrong image | plan prints the image (L24.5) | the stage's | the image digest | the source mount's `image=` |
| dependency version outside the declared range | plan (L24.6) | 3 | the declared range and `/sys` version | change the range or the target |
| undeclared egress | plan (L24.10); flows counted at runtime | 0 with a warning | hosts dropped, from `flows` | `--egress=<host>` capped by the mount |
| defaulted limit exceeded | OOMKilled | 1 for the replica | the limit and its source (LimitRange) | `--memory` on the unit |
| digest without the mount's provenance | plan (L24.9) | 3 | builder, attestation, secret-shaped files | release CI's digest |
| profile difference crashes production | the diff at `release` (L24.4) | 3 under `--strict`, else a warning | each difference | fix the image or set the test mount like production |

## 8. Acceptance criteria

- **AC-24.1** Given `run --unit` or `release` with an image, then `explain` resolves the digest's platforms and refuses one that no node in the target pool can run, naming both.
- **AC-24.2** Given a unit, then its environment is only what the unit declares, `explain` lists each variable with its source, and the session's exported variables reach it only if named with `--env NAME`.
- **AC-24.3** Given `explain --at=<mount>` or `release --to=<mount>`, then the plan for the target is printed in a canonical order and compared with the mount where the digest last passed, listing every difference in profile, Pod Security, filesystem, limits, egress, node platform, dependency versions, `caps` and Kubernetes version.
- **AC-24.4** Given a source-repository mount with `image=`, then local stages under that path run in that image and `explain` names it; without it, `explain` prints `runs in toolbox <digest>`.
- **AC-24.5** Given `--connect=<path>@<range>`, then the plan reads the target's version from `/sys` and refuses outside the range.
- **AC-24.6** Given an image with no egress annotation, then `explain` prints `egress: undeclared`, the unit may declare `--egress`, and dropped connections are counted per unit where `caps: flows` exists.
- **AC-24.7** Given a target namespace, then `explain` prints its defaulted requests and limits, its quota headroom and its Pod Security level, each with the object that sets it.
- **AC-24.8** Given a mount with `provenance=ci`, then a digest without the platform's build attestation is refused at plan time with the CI path, and a digest with secret-shaped files in its layers is refused on every mount under the scrub rules of ADR 0007 amendment 2.
- **AC-24.9** Given a unit started from a branch other than the default, then it carries the profile's `branch_unit_ttl`, is listed with its daily cost, and ends at the TTL or when the branch is deleted, with a receipt.

## 9. Persona lenses

- **DEV**: wants to keep the laptop loop, to know the image their tests run in, and to read one list of differences before production instead of one crash per difference.
- **DO**: wants production to take only CI's digests and to say so at plan time, not at admission after an approval was spent.
- **SRE**: wants a release to print what is different from where the digest last passed, because that is where the incident comes from.
- **PE**: wants the namespace's own defaults, the ones the platform set by LimitRange and Pod Security labels, to appear in `explain`, and branch units to end on their own.
- **SEC**: wants secret-shaped files in image layers caught before any mount runs them, and laptop builds kept out of production.
- **FIN**: wants branch units priced per day where the developer sees them, and ended when the branch is.

## 10. Findings

| Id | Probe | Finding | Proposed disposition | Touches |
|---|---|---|---|---|
| L24.1 | P11 | The laptop is a terminal (ADR 0005), so the working tree a developer edits there has no path in the session; the only route is pushing unfinished code to a shared branch, and nothing covers a locally built image either. | **open** (drafted 2026-11-11 in [ADR 0022](../adr/0022-the-laptop-lends-a-tree-the-cluster-builds-it.md): the laptop lends a working tree as snapshots filtered and scrubbed on the laptop, the cluster builds with `image-build` under `provenance: session`, and a dev unit runs with `--like`, `--follow` and `--debug`, reached by `attach -L`): the inner loop needs a design: a read-only sync of a declared directory over the attach channel (ADR 0020 decision 5) as a device `/dev/scm/by-id/wt-<host>` in the session's residency, with the same scrub as artifacts, or branches only; and how a locally built image reaches a test registry | ADR 0005; ADR 0011 a2; ADR 0020 d5; S6 |
| L24.2 | P7 | `explain` resolves a tag to a digest but never checks that the digest has a platform the target nodes run; an arm64-only image passes the plan and crash-loops on amd64. | **rule** (applied: ADR 0013 rules; roadmap M8): `run --unit`, `run --image` and `release` read the digest's platform list from the registry and the target pool's platforms from `/sys`; no match is refused at plan time naming both, and a partial match pins placement to the matching nodes and says so | ADR 0013 d3, d4; ADR 0011 d7 |
| L24.3 | P12 | ADR 0005 makes every launched process inherit the session's env. A unit outlives the session and is re-created by every release, so a developer's `export` becomes the service's configuration in test and, through a release, in production. | **amend** (applied: ADR 0013 d4 revised; ADR 0005 note; command map) ADR 0013 decision 4: a unit's environment is its declaration only (`--env NAME=value`, `--env NAME=ref:<path>`, `--env NAME` to copy one session variable by name), recorded in the unit record and the GitOps intent; session env reaches a unit only when named; `explain` lists each variable with its source; processes other than units keep inheriting under ADR 0005 | ADR 0013 d4, d7; ADR 0005; ADR 0001 (env subset); S2 |
| L24.4 | P6, P8 | "Green in test" is a result computed under one environment and reused for another; `explain` prints one mount's plan at a time, so the six ways test differs from production (Pod Security, read-only root, limits, egress, provenance, Kubernetes version) surface as six crashes. | **amend** (applied: ADR 0013 d3 and d7 revised; command map) ADR 0013 decision 3: `explain --at=<mount>` prints the plan a command would have under another mount without `cd`, in a canonical order, so `diff` of two plans lists their differences; `release --to=<mount>` prints the differences from the mount where the digest last passed its readiness (from the unit receipts) and refuses under `--strict` or the target profile's `release_diff=refuse` | ADR 0013 d3, d7; ADR 0010; ADR 0017 (the diff is shown in the approval) |
| L24.5 | P2 | A local stage runs in the session's toolbox (ADR 0003), so `make test` in a source tree uses the toolbox's Python, not the service's; a test result in the session says nothing about the image. | **rule** (applied: ADR 0003 rules; roadmap M3; command map): a source-repository mount takes `image=<ref>\|devcontainer` (the repository's `.devcontainer` definition); local stages whose cwd is under that mount run in that image as a parallel-local backend pod, and `explain` names the image of every stage, `toolbox <digest>` included | ADR 0003; ADR 0011 a2; ADR 0013 d3 |
| L24.6 | P9 | `--connect` declares reach to a dependency but not which version the service needs; the test database's PostgreSQL 16 against the laptop's 17 surfaces as a failed migration. | **rule** (applied: ADR 0019 rules; roadmap M11): `--connect=<path>@<range>` checks the version the device reports in `/sys` at plan time and refuses outside the range; the unit record keeps the range, so `release` checks it on every target | ADR 0019 d3; ADR 0011 d7; ADR 0015 |
| L24.7 | P9 | The namespace's LimitRange, ResourceQuota and Pod Security labels are set by the platform outside the shell; `explain` shows none of them, so a laptop with no limit meets a 512 Mi default as an OOM kill. | **rule** (applied: ADR 0013 rules; roadmap M8): `explain` of any plan into a namespace prints its defaulted requests and limits, its quota headroom and its Pod Security level, each with the object that sets it, and warns when a unit declares none and inherits a default | ADR 0013 d3; ADR 0010; ADR 0003 |
| L24.8 | P3, P5 | Units started from branches keep running after the branch merges; nothing ends a unit except `kill`, and nobody is told the daily cost. | **rule** (applied: ADR 0013 rules; roadmap M9; command map): a unit whose image or source comes from a branch other than the default carries the profile's `branch_unit_ttl`; `ps` and `cost` show its daily cost; it ends at the TTL or when the branch is deleted, with a receipt and a notice to the owning group | ADR 0010; ADR 0013 d7; ADR 0006 |
| L24.9 | P4 | A digest built and pushed from a laptop can be released to production; the cosign check at admission refuses it only after the approval is spent, and nothing looks at what the layers hold (a `.env` copied by `COPY . .`). | **rule** (applied: ADR 0013 rules; roadmap M7): a mount's profile names the provenance it requires (`provenance=ci\|any`); `explain` and `release` read the digest's attestation and refuse at plan time with the CI path; on every mount, a digest whose layers hold files that match the artifact scrub of ADR 0007 amendment 2 is refused, naming the file and the rule, never the value | ADR 0010; ADR 0013 d7; ADR 0007 a2 |
| L24.10 | P7 | Egress is declared by the executable's annotation (ADR 0019 decision 4); a team's own image has none, so under `egress=declared` it may reach nothing, and its calls fail as timeouts with no notice. | **rule** (applied: ADR 0019 rules; roadmap M11; command map): an image with no annotation has `egress: undeclared`, printed by `explain` with the mount's default; a unit declares `--egress=<host>…` in its record, capped by the mount as an annotation would be; where `caps: flows` exists, dropped connections are counted per unit and shown by `netstat` with the host | ADR 0019 d4; ADR 0002; ADR 0013 d4 |
| L24.11 | none | A race that only production traffic shows cannot be debugged interactively: no debugger attaches to a production process, and the cluster never calls back into the laptop. | **accepted**: the laptop is a terminal and production processes are not inspected without break-glass (rule L21.9 applies the same principle to sessions); logs, `attach -L` to owned units and a test mount set like production through the L24.4 diff are the route | ADR 0005; ADR 0019 d7; ADR 0016 d6 |
