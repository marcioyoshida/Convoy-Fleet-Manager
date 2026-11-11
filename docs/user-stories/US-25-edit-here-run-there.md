# US-25 · Edit here, run there: lend my monorepo to the cluster, reload and debug a dev unit, and ship the exact tree I tested

| Field | Value |
|---|---|
| Primary persona | DEV |
| Secondary personas | SEC, PE, FIN, SRE, DO |
| Workload class | the inner loop of one service in a monorepo: a lent working tree, builds in the cluster, a followed and debugged dev unit in a shared test namespace, and the commit that leaves the laptop afterwards |
| Walkthrough | written against [ADR 0022](../adr/0022-the-laptop-lends-a-tree-the-cluster-builds-it.md) (all ten decisions), with [ADR 0020](../adr/0020-a-session-is-a-journal-not-a-pod.md) decision 5 (the attach protocol), [ADR 0011](../adr/0011-device-tree-and-mount-table.md) amendments 2 and 4 (source repositories, `disconnected`), [ADR 0007](../adr/0007-secrets-are-references.md) amendment 2 (the scrub), [ADR 0013](../adr/0013-native-verbs-have-one-shape.md) decisions 3, 4 and 7 with rules L24.4, L24.8 and L24.9, [ADR 0019](../adr/0019-a-unit-has-an-address-reach-is-a-grant.md) decision 7 (`attach -L`), [ADR 0016](../adr/0016-identity-is-a-user-rights-belong-to-groups.md) decisions 3, 5 and 11 and [ADR 0010](../adr/0010-platform-defaults-and-two-package-tiers.md) amendment 1 (intent); it continues US-24 |
| Gaps already closed there | none |
| Milestones needed | M2, M3, M4, M5, M7, M11 |
| Findings in this document | L25.1 – L25.16 |

## 1. Story

**As an** application developer, **I want** to keep editing on my laptop and see each change running in the cluster's test environment within seconds, step through it in my own debugger, and then ship exactly the code I tested, **so that** the inner loop is as fast as `docker compose` without the arm64 image, the leaked `.env` or the "works on my machine" that US-24 ended in, and without my experiments breaking my teammates' tests.

**Context.** Two weeks after US-24, the team has ADR 0022 in its test profile. `orders-api` has moved into the company monorepo `acme/platform` (2.4 GiB, 61 000 files: `services/orders-api`, `services/billing`, `libs/money`, `tools/`, `data/`), and its Dockerfile copies `libs/money` from two levels up. `kai` (user `kai`, group `orders`) works on an arm64 laptop called `mbp`; so does their teammate `noor`, whose `billing-api` in test calls `orders-api`. The test mount `/clusters/test/orders` has profile `test`: `provenance=session`, debugging allowed, `release_path=git` because the namespace is synced by Argo CD with self-heal and prune, a shared PostgreSQL cluster, `branch_unit_ttl=3d`, `lend_max=1Gi`, `lend_file_max=50Mi`, `worktree_retention=7d`. The session's residency is `us`; production `prod-eu` is in `eu`.

## 2. Actors and identities

| Role | Who | Identity Convoy sees | Notes |
|---|---|---|---|
| Runs it | `kai` | user `kai`, group `orders` | lends from the laptop, debugs from the editor |
| Shares the environment | `noor` | user `noor`, group `orders` | `billing-api` and an end-to-end suite in test |
| Builds | `image-build` | `svc:orders` | rootless BuildKit in the session's residency; pushes to the group's development path |
| Builds for production | CI | `svc:orders` through the CI's trust policy | attests `provenance: ci` |
| Pays | `orders` | snapshots, build cache, builds, dev units | |
| Operates the platform | platform team | the lend defaults, the scrub rule set, the test and prod profiles, Argo CD on the test namespace, prod-eu's mesh and admission policies | |

## 3. Preconditions

- **Capabilities** (`caps`): `dev-registry`, `image-build`, `scm:github`, `pod-security`, `flows`; in prod-eu also `mesh:istio` and `admission:kyverno`.
- **Packages**: the toolbox with `image-build`; debug agents in the team's base image.
- **Mounts**: `/clusters/test/orders`, `/clusters/prod-eu/orders`; the lent trees appear under `/src/`.
- **Annotations**: the base image carries `live: sync services/orders-api/orders/ then kill -HUP 1` and `debug: debugpy 5678`.
- **Platform defaults** `explain` must print: `lend_max`, `lend_file_max`, the excluded patterns and scrub rule version, `worktree_retention`, the provenance each mount accepts, `debug` per profile, `branch_unit_ttl`, the mount's `release_path`.

## 4. Main path

```bash
# Monday 09:10: lend the service directory
laptop$ convoy attach --lend=~/src/platform/services/orders-api:orders
lend orders: 412 files, 3.1 MiB, tree t:5be0…; withheld 1 (config/local.yaml: rule stripe-key)
[kai] $ cd /clusters/test/orders
[kai] $ run image-build /src/orders
step 4/11 COPY ../../libs/money /app/libs/money: path outside the build context                   (L25.1)

# lend the monorepo instead
laptop$ convoy lend ~/src/platform:platform
refused: 2.4 GiB above lend_max 1 GiB; largest data/ 1.9 GiB, services/billing/node_modules 210 MiB
#  kai adds data/ and node_modules/ to .convoyignore
lend platform: 18 204 files, 310 MiB, tree t:9a71…; withheld 2
#  sent among them: services/orders-api/tests/fixtures/prod-sample.sql, 38 MiB of prod-eu orders
#  with customer e-mail addresses, now in the us artifact store for seven days                     (L25.3)
[kai] $ run image-build /src/platform --file=services/orders-api/Dockerfile
step 6/11 COPY services/orders-api/gen/ /app/gen/: not found
#  gen/ is generated by protoc and git-ignored, so the lend left it out; the laptop build has it;
#  config/local.yaml was withheld, and step 8 copies it too; the plan said neither              (L25.2)
[kai] $ run image-build /src/platform --file=services/orders-api/Dockerfile
step 7/11 RUN pip install -r requirements.txt
#  a dependency's setup.py runs as svc:orders, inside a build that holds the dev-registry push token
#  and the group's egress                                                                          (L25.9)
digest sha256:41c2…  provenance: session  tree t:9a71…

# 10:05: run it where it will run
[kai] $ run --unit orders-api --from=/src/platform --follow --debug=5678
unit orders-api updated: image sha256:41c2…, 1 replica, debug 5678
#  orders-api is the team's shared unit; noor's billing-api now calls kai's debug pod             (L25.7)
#  three minutes later Argo CD self-heals the Deployment back to release 1.4.2
[kai] $ run --unit orders-api-kai --from=/src/platform --follow --debug=5678
unit orders-api-kai: 1/1 ready
#  six minutes later Argo CD prunes it: the namespace holds only what git declares               (L25.12)

# 10:40, in kai's own development namespace with release_path=direct
[kai] $ cd /clusters/test/orders-dev
[kai] $ run --unit orders-api-kai --from=/src/platform --follow --debug=5678 --connect=/clusters/test/orders/db
laptop$ convoy attach -L 5678:orders-api-kai-0:5678         # the editor's debugger attaches
#  kai stops at a breakpoint inside a refund transaction and goes to lunch;
#  noor's migration test waits 52 minutes on the row lock in the shared database                (L25.6)
#  every save under orders/ syncs and HUPs; each edit of requirements.txt rebuilds and rolls out:
#  31 builds before 15:00, $11.40, none of it on kai's screen                                     (L25.8)

[kai] $ cd /src/platform/services/orders-api && make test
OSError: [Errno 30] Read-only file system: '.pytest_cache'                                        (L25.13)

laptop$ convoy attach --env-of=orders-api-kai > .env.test
#  STRIPE_API_KEY=sk_test_51Hx… is in the file: it was declared --env STRIPE_API_KEY=<value>
#  on Friday, not as a reference                                                                  (L25.14)

[kai] $ run --unit orders-api-kai … --like=/clusters/prod-eu/orders
like prod-eu: Pod Security restricted, read-only root, limits, egress applied
#  prod-eu also has an Istio AuthorizationPolicy and a Kyverno mutation the platform's GitOps
#  added; neither is reproduced and nothing says so                                              (L25.15)

[noor] $ ls /dev/scm/by-id/ | grep wt-
wt-mbp-platform
#  kai's uncommitted tree, NOTES.md included, readable by all of orders; noor's laptop is also
#  called mbp, and lending its platform tree would take the same id                              (L25.10)

# 15:20: the bug is fixed in the running pod; noor asks which digest to test against
[kai] $ ps | grep orders-api-kai
orders-api-kai  sha256:41c2…  running  2d23h left  $3.10/day
#  the pod runs sha256:41c2… plus 14 synced updates (tree t:c3d9…); noor's e2e run with
#  --image=sha256:41c2… fails on the bug kai fixed; the unit receipt says 41c2 passed readiness
#  in test, which is what the release diff of L24.4 will compare against                         (L25.5)

# 18:40: kai closes the lid during a save
# Tuesday 08:30
[kai] $ cat /src/platform/services/orders-api/orders/refund.py
cat: I/O error: tree t:e07f… is current but refund.py never arrived; device disconnected          (L25.4)

# Tuesday 11:00: ship it
laptop$ git commit -am "refund rounding" && git push
#  CI builds sha256:77ab… from commit 5e1c…; nothing ties 5e1c… to t:c3d9…, the tree that passed  (L25.11)
#  save-to-reload took 3 s at the office and 41 s on the train; nothing promised either            (L25.16)
```

| Step | Convoy object | Kubernetes or cloud object | Where the state lives |
|---|---|---|---|
| `attach --lend` | the lend channel; device `/dev/scm/by-id/wt-…` mounted at `/src/<name>` | none | the client's manifest; snapshots in the artifact store, class `worktree` |
| `image-build` | an executable, `class: build`, `side effect: push` | a rootless BuildKit pod as `svc:orders`; a push to the registry's development path | the layer cache in the artifact store; the attestation in the receipt |
| `run --unit --from --follow --debug` | a dev unit, a branch unit under L24.8 | Deployment, Service, the debug port, a relaxed liveness probe | the unit record and receipt lines per tree id |
| `attach -L` | a forward on the attach protocol | a port-forward to the pod | the session journal |
| `attach --env-of` | a file written by the client | none | the laptop |

## 5. Abstraction coverage

| # | Aspect | Touched | How this story uses it | What to verify |
|---|---|---|---|---|
| A1 | Pipe: staged artifact, content id, cache | ● | a snapshot is content; a followed unit's code is not its digest (L25.5); a tree id that a commit can match (L25.11) | AC-25.5, AC-25.11 |
| A2 | Command: class, annotation, executable fields | ○ | `image-build` and the `live:` and `debug:` annotations | |
| A3 | Session: mount table, env, cwd, attach and detach | ● | the lend channel, `--env-of`, the laptop sleeping mid-save (L25.4, L25.14) | AC-25.4, AC-25.14 |
| A4 | Backend selection | ● | a local stage under a read-only worktree (L25.13); a build in a pod | AC-25.13 |
| A5 | Completion contract | ○ | a dev unit's readiness as the exit of `run --unit` | |
| A6 | Device tree: `/dev`, mounts and options, `self`, `all`, `/sys` | ● | the `worktree` device, its id and who may read it (L25.10); a monorepo as one tree (L25.1) | AC-25.1, AC-25.10 |
| A7 | Signals and lifecycle | ● | a debugger pause is a stop no signal sent and nobody bounds (L25.6); `live: … then kill -HUP 1` | AC-25.6 |
| A8 | Secrets as references | ● | a withheld file the build needs (L25.2); a plain env value written to the laptop (L25.14) | AC-25.2, AC-25.14 |
| A9 | Cost | ● | rebuilds triggered by saves (L25.8) | AC-25.8 |
| A10 | Identity and authorization | ● | lent code runs inside a build that holds the group's push token (L25.9); a snapshot readable by the whole group (L25.10) | AC-25.9, AC-25.10 |
| A11 | Platform defaults and `caps` | ● | `lend_max`, the exclusions, `release_path`, what `--like` cannot copy (L25.3, L25.12, L25.15) | AC-25.3, AC-25.12, AC-25.15 |
| A12 | Storage | ○ | snapshots and build cache under `worktree_retention` | |
| A13 | Networking | ● | a dev unit that took a shared unit's address (L25.7); build egress (L25.9) | AC-25.7 |
| A14 | Time and queueing | ○ | the branch-unit TTL; a pause with no bound; reload latency (L25.16) | |
| A15 | Audit, receipts, retention and `replay` | ○ | receipt lines per followed update | |
| A16 | Durability | ● | a snapshot that is current but incomplete when the laptop sleeps (L25.4) | AC-25.4 |

## 6. Loophole probes

| # | Probe | Question for this story | Design answer | Finding |
|---|---|---|---|---|
| P1 | Imperative verb, declarative owner | a dev unit created directly in a namespace Argo CD syncs with self-heal and prune | ADR 0022 decision 6 creates the unit "as any unit"; nothing checks the mount's `release_path` | **L25.12** |
| P2 | POSIX semantics that cannot hold | `make test` under `/src/<name>` writes caches into the tree | the worktree is mounted read-only (decision 1) | **L25.13** |
| P3 | Orphaned or lost state | snapshots after the lend ends; dev units after the day ends | `worktree_retention` and `branch_unit_ttl` (decision 3, L24.8) | ok |
| P4 | Identity leak or escalation | (a) code from the laptop runs in a build holding `svc:orders`'s push token and egress; (b) one developer's uncommitted tree is readable by the whole group, under an id another laptop can collide with | (a) decision 4 builds as `svc:<group>` and says nothing about the build steps; (b) decision 3 makes the snapshot the primary group's | **L25.9**, **L25.10** |
| P5 | Invisible cost | `--follow` rebuilds on every save that touches anything outside `live: sync` | priced to the group (Consequences), but shown nowhere the developer looks while it happens | **L25.8** |
| P6 | Lying cache | (a) a followed unit's code is its digest plus synced updates, and its receipts say the digest passed; (b) the commit CI builds has no recorded relation to the tree that passed | (a) decision 7 writes the tree id in the receipt but `ps`, `--image` and the release diff use the digest; (b) the tree id is the lend's own hash | **L25.5**, **L25.11** |
| P7 | Silent failure | the build fails late on a context path outside the lent directory, a git-ignored generated directory and a withheld file | the lend reports a count of withheld files; the build learns the rest | **L25.1**, **L25.2** |
| P8 | Partial fleet | the laptop sleeps between publishing a tree id and sending its files | decision 1 publishes snapshots; decision 2 sends files on request, so a current tree can be incomplete | **L25.4** |
| P9 | Created elsewhere | `--like=prod-eu` and the policies the platform put on prod-eu outside Convoy | decision 6 applies what the profile and `explain --at` know | **L25.15** |
| P10 | Multi-party action | (a) `run --unit orders-api --from` on the team's shared unit; (b) a breakpoint holding a lock in the shared database | (a) a unit name is a unit; nothing distinguishes a dev unit's name; (b) decision 8 relaxes liveness while a debugger is attached, with no bound | **L25.7**, **L25.6** |
| P11 | Data at rest | a production data sample in the working tree, lent to an artifact store in another residency | the filters are for secrets; the scrub does not classify data; the snapshot lands in the session's residency | **L25.3** |
| P12 | Process-tree inheritance | `--env-of` copies a unit's env to the laptop | values that are references are names only (decision 9); a secret declared as a plain value is "non-secret" | **L25.14** |

A thirteenth question: ADR 0022 leaves the latency from save to a followed unit's reload open. On the attach protocol it is the WAN round trip plus the debounce plus a copy and a HUP, or a build: seconds at the office, tens of seconds on a train. **L25.16**, accepted.

## 7. Failure matrix

| Failure | Detected by | Exit | Receipt says | Recovery verb |
|---|---|---|---|---|
| build context outside the lent tree | plan (L25.1) | 3 | the path and the lend's root | lend the root with `--only` |
| a COPY of an excluded or withheld file | plan (L25.2) | 3 | each file and the rule or ignore file that left it out | re-include in `.convoyignore`, or remove the file from the build |
| a data file in the lend | the client's filter (L25.3) | 3 | the file, the pattern and the residency | `.convoyignore`, or `--with-data` where the profile allows |
| snapshot incomplete at disconnect | the tree is not published until complete (L25.4) | none | the last complete tree and its age | reattach |
| a followed unit's code differs from its digest | `ps`, `explain` (L25.5) | 3 for `--image` or release of the digest under `--strict` | digest, tree, updates since build | `image-build` from the tree |
| a debugger paused past the bound | the debug agent's report (L25.6) | none | pause start, holder, restart | reattach the debugger |
| a dev unit name taken by a shared unit | plan (L25.7) | 3 | the shared unit and its owner | the default dev name |
| rebuild rate above the cap | the follow loop (L25.8) | none | builds coalesced, cost so far | `--follow=sync` |
| a dev unit in a reconciled namespace | plan (L25.12) | 3 | the mount's `release_path` and the reconciler | `cd` to a dev mount |
| writes under a read-only worktree | the overlay (L25.13) | the stage's | the overlay and what was discarded | redirect outputs |
| a secret-shaped plain env value | plan (L25.14) | 3 | the variable name and the rule, never the value | `--env NAME=ref:<path>` |

## 8. Acceptance criteria

- **AC-25.1** Given `lend <root> --only=<path>…`, then one snapshot holds those paths under one tree id, and `image-build` refuses at plan time a context or `COPY` source outside the snapshot, naming the path and the lend's root.
- **AC-25.2** Given `explain lend <dir>` or `lend --dry-run`, then the client prints what it would send, exclude and withhold with sizes and the rule or ignore file for each; and given `image-build`, then the plan lists every context file excluded or withheld and refuses when a `COPY` or `ADD` needs one.
- **AC-25.3** Given a file that matches the platform's `lend_exclude` data patterns, then the client refuses the lend naming the file, the pattern and the session's residency, unless `--with-data` is given and the profile allows it.
- **AC-25.4** Given a lend interrupted after some files were sent, then the device's current tree id is still the last complete snapshot, and the partial one is never readable.
- **AC-25.5** Given a followed unit, then `ps` and `explain` show its digest, its current tree and the number of updates since the build; `--image=<digest>` and the release diff treat the digest as passed only for the tree it was built from.
- **AC-25.6** Given `--debug`, then `ps` shows a paused process with its pause duration, and past the profile's `debug_pause_max` the pod is restarted with a receipt line.
- **AC-25.7** Given `run --unit <name> --from`, then a name already held by a unit the user did not create from a lend is refused, the default name is `<name>-<user>`, and no Service selector of a shared unit ever matches a dev unit.
- **AC-25.8** Given `--follow`, then at most one build runs per unit, later saves coalesce into the next, `follow_rebuild_max` per hour stops rebuilding with a notice, and `ps` shows the builds and their cost so far.
- **AC-25.9** Given `image-build`, then build steps run with no registry credentials and with the profile's `build_egress`, and the push is made by the builder after the last step.
- **AC-25.10** Given a lend, then the device id carries the user (`wt-<user>-<host>-<name>`), and its snapshots are readable by the lender only unless `--share=<group>` is given.
- **AC-25.11** Given a snapshot of a git working tree, then its tree id is the git tree hash of the lent paths, and `release` and `explain` of a CI digest print the snapshot that passed with the same tree, or say none did.
- **AC-25.12** Given `run --unit --from` in a mount whose `release_path` is `git`, then the plan is refused naming the reconciler and the platform's development mounts.
- **AC-25.13** Given a local stage whose cwd is under a worktree mount, then it gets a per-process writable overlay discarded at exit, and `explain` says so.
- **AC-25.14** Given a unit `--env NAME=<value>` whose value matches the scrub rules, then the plan is refused; and `attach --env-of` writes no value that matches them.
- **AC-25.15** Given `--like=<mount>`, then `explain` lists the policies on the target that `--like` does not reproduce (mesh, admission mutations, objects outside the profile), each with its kind and name.

## 9. Persona lenses

- **DEV**: wants the lend to say before the build what it left out, a dev name that cannot hit the team's unit, and one line that says whether the CI digest is the code they tested.
- **SEC**: wants production data stopped on the laptop, build steps without the group's push token, a snapshot visible only to its author, and no secret-shaped value on a laptop under any flag.
- **PE**: wants dev units only in namespaces nobody reconciles, the lend's defaults printed, and `--like` honest about what it cannot copy.
- **FIN**: wants the cost of a followed unit's rebuilds on the developer's screen while it accrues, and a cap.
- **SRE**: wants a forgotten breakpoint bounded, because a shared database lock is an incident in test today and a habit in production tomorrow.
- **DO**: wants the commit CI builds tied to the tree that passed, so "tested" means the bytes that ship.

## 10. Findings

| Id | Probe | Finding | Proposed disposition | Touches |
|---|---|---|---|---|
| L25.1 | P7 | A lend is one directory, and a monorepo service's build context reaches outside it (`COPY ../../libs/money`); lending the whole monorepo exceeds `lend_max`. The build fails late on a missing path. | **amend** ADR 0022 decision 1: `lend <root> --only=<path>…` lends a sparse set of paths of one root as one snapshot with one tree id; a repository's `.convoylend` may declare the paths per service; `image-build` refuses at plan time a context or `COPY` source outside the snapshot, naming the path and the root. Settles ADR 0022's open item on monorepos | ADR 0022 d1, d4 |
| L25.2 | P7 | The lend reports only a count of withheld files. A generated directory left out by `.gitignore` and a withheld config file are both needed by the Dockerfile, so the laptop build works and the cluster build fails, which is "works on my machine" again. | **rule** (ADR 0022 rules; command map): `explain lend <dir>` and `lend --dry-run` print what would be sent, excluded and withheld, with sizes and the rule or ignore file for each; `image-build` lists the excluded and withheld files under its context and refuses when a `COPY` or `ADD` needs one; `.convoyignore` may re-include a git-ignored path (`!gen/`), which is still scrubbed | ADR 0022 d2, d4; ADR 0007 a2 |
| L25.3 | P11 | The filters are for secrets. A production data sample in a test fixture is lent as an ordinary file and stored for seven days in the session's residency, which is not production's. | **rule** (ADR 0022 rules; ADR 0010 defaults): the platform default `lend_exclude` names data patterns (`*.sql`, `*.dump`, `*.parquet`, `*.sqlite`, `*.csv` above a size) that the client refuses, naming the file, the pattern and the session's residency; `--with-data` overrides only where the profile sets `lend_data=allow`; the lend's receipt lists the data files sent | ADR 0022 d2, d3; ADR 0010; ADR 0016 d11 |
| L25.4 | P8 | Decision 1 publishes a snapshot's tree id; decision 2 sends files when the session requests them. A laptop that sleeps between the two leaves a current tree whose files never arrived, which `disconnected` does not describe. | **amend** ADR 0022 decisions 1 and 2: a snapshot becomes current only when every file in its manifest, less the withheld ones, is in the artifact store; the session's request by path is an authorization (it may name only manifest paths), not a lazy fetch; an interrupted snapshot is discarded and the device stays on the last complete tree, with its age | ADR 0022 d1, d2, d3; ADR 0011 a4; ADR 0020 d5 |
| L25.5 | P6 | A followed unit runs its digest plus synced updates. `ps`, `--image` and the receipts that the release diff (L24.4) reads all name the digest, so the digest is recorded as passing for code it does not contain. | **rule** (ADR 0022 rules; ADR 0013 rules): a followed unit's identity is digest and tree; `ps` and `explain` show both and the number of updates since the build; a digest counts as passed in a mount only for the tree it was built from, so `--image=<digest>` and the release diff refuse under `--strict` and otherwise warn when the passing run carried live updates | ADR 0022 d7; ADR 0013 d7; L24.4 |
| L25.6 | P10 | A debugger paused at a breakpoint is a stop nobody sent and nothing bounds. Liveness is relaxed while it is attached, so a forgotten breakpoint holds a transaction lock in a shared database for as long as the laptop is away. | **rule** (ADR 0022 rules; ADR 0010 defaults): the debug agent reports pause and resume; `ps` shows the process as `paused (debugger)` with its duration; past the profile's `debug_pause_max` (default 10 minutes) the pod is restarted, with a receipt line naming the pause and the connections it held | ADR 0022 d8; ADR 0008; ADR 0010 |
| L25.7 | P10 | `run --unit orders-api --from` on a name the team's shared unit holds updates the shared unit, so its callers reach a paused debug process. A dev unit's name and address are not kept apart from the units others rely on. | **amend** ADR 0022 decision 6: a dev unit is named `<name>-<user>` by default; a name held by a unit not created from a lend is refused, naming the unit and its owner; a dev unit gets its own address, and no Service of another unit selects it; callers reach it only by an explicit `--connect` | ADR 0022 d6; ADR 0019 d2, d3; ADR 0013 d4 |
| L25.8 | P5 | `--follow` rebuilds on every save outside the `live: sync` paths. Thirty-one builds in a day cost money that appears in `cost` but nowhere the developer is looking. | **rule** (ADR 0022 rules; ADR 0010 defaults): one build in flight per followed unit, later saves coalescing into the next; `follow_rebuild_max` per hour, after which the unit stays on sync-only updates with a notice; `ps` shows the builds and their cost since `--follow` began; `--follow=sync` never rebuilds | ADR 0022 d7; ADR 0010 |
| L25.9 | P4 | Code lent from a laptop, including its dependencies' install scripts, runs in a build as `svc:orders` that holds the development registry's push token and the group's egress. | **rule** (ADR 0022 rules; ADR 0019 rules): build steps run with no registry credentials except a pull-only token for the base images, and with the profile's `build_egress` (package mirrors by default); the builder pushes after the last step with its own credential; the attestation records the egress used | ADR 0022 d4; ADR 0016 d5; ADR 0019 d4 |
| L25.10 | P4 | A snapshot belongs to the session's primary group, so every member reads a colleague's uncommitted tree; the device id `wt-<host>-<name>` collides when two laptops share a host name. | **rule** (ADR 0022 rules; ADR 0011 rules): the device id is `wt-<user>-<host>-<name>`; snapshots are readable by the lender and the build and units they start, and by a group only with `lend --share=<group>`, which is recorded | ADR 0022 d1, d3; ADR 0016 d3; ADR 0011 a2 |
| L25.11 | P6 | The commit CI builds has no recorded relation to the snapshot that passed in test, so "tested" and "shipped" are linked by the developer's memory. | **amend** ADR 0022 decision 1: for a git working tree, a snapshot's tree id is the git tree hash of the lent paths (withheld files hashed as their working-tree content on the laptop, never sent); the commit is made on the laptop, never in the session; `explain` and `release` of a CI digest print the snapshot that passed with the same tree, or say that none did. Settles ADR 0022's open item on promoting a snapshot | ADR 0022 d1, d5; ADR 0001; ADR 0013 d7 |
| L25.12 | P1 | A dev unit created directly in a namespace Argo CD syncs is reverted by self-heal or deleted by prune within minutes; the developer sees a unit that worked and then did not. | **rule** (ADR 0022 rules; ADR 0010 rules): `run --unit --from` is refused in a mount whose `release_path` is `git`, naming the reconciler and the platform's development mounts (`dev_mounts`); a development mount has `release_path=direct` and is excluded from the reconciler | ADR 0022 d6; ADR 0010 a1 |
| L25.13 | P2 | The worktree is read-only, so a local stage under it (`make test`) fails on its own caches. | **rule** (ADR 0003 rules; ADR 0022 rules): a local stage whose cwd is under a worktree mount gets a per-process writable overlay, discarded at exit; outputs meant to be kept are redirected to artifacts; `explain` prints `overlay: discarded` | ADR 0022 d1; ADR 0003; L24.5 |
| L25.14 | P12 | `--env-of` writes the unit's non-secret variables to the laptop, and "non-secret" means declared as a value. A key declared as a plain value is written to a file on the laptop. | **rule** (ADR 0013 rules; ADR 0022 rules): a unit's `--env NAME=<value>` whose value matches the scrub rules is refused at plan time naming the variable and the rule; `attach --env-of` applies the same rules and writes no matching value | ADR 0022 d9; ADR 0013 d4; ADR 0007 a2 |
| L25.15 | P9 | `--like=prod-eu` reproduces what the profile and `explain --at` know. The mesh authorization policy and the admission mutation the platform added to prod-eu are not reproduced, and nothing says so. | **rule** (ADR 0022 rules; ADR 0013 rules): `explain` of `--like` lists the policies on the target that it does not reproduce, by kind and name (mesh policies, admission mutations, objects outside the profile), read from `caps` and the target namespace | ADR 0022 d6; ADR 0013 d3; L24.7 |
| L25.16 | none | Save-to-reload latency over the attach protocol is the WAN round trip plus the debounce plus a sync or a build; no target holds on every link. | **accepted**: no latency target; `explain` of a followed unit prints the measured save-to-ready time of its last updates, so the developer can see when a train ride makes the cluster loop slower than the laptop's | ADR 0022 d7; ADR 0020 d5 |
