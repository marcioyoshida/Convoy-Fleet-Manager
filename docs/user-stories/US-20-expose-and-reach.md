# US-20 · Put an API on the internet, let one other team call it, and reach a broken pod from my laptop without opening anything else

| Field | Value |
|---|---|
| Primary persona | SRE |
| Secondary personas | DO, DS, SEC, PE, FIN |
| Workload class | service units: a model API at three reaches and across two regions, a cross-team grant, a notebook for a person's browser, interactive debugging from the session and the laptop, a cluster whose CNI enforces nothing, a canary under long-lived streams |
| Walkthrough | written against [ADR 0019](../adr/0019-a-unit-has-an-address-reach-is-a-grant.md) with [ADR 0016](../adr/0016-identity-is-a-user-rights-belong-to-groups.md) (groups as tenants), [ADR 0017](../adr/0017-approvals-are-objects-on-the-process.md) (the `--public` gate), [ADR 0018](../adr/0018-labels-are-the-signal-admission-is-the-decision.md) (labels, slugs, mappings), [ADR 0010](../adr/0010-platform-defaults-and-two-package-tiers.md) (amendment 1, rule L7.6), [ADR 0011](../adr/0011-device-tree-and-mount-table.md) (`caps`, residency) and [ADR 0013](../adr/0013-native-verbs-have-one-shape.md) (`release --canary`); it reuses the `churn-api` and notebook of US-04 and the release of US-03 |
| Gaps already closed there | none |
| Milestones needed | M7, M8, M9, M11 |
| Findings in this document | L20.1 – L20.11 |

## 1. Story

**As an** SRE who runs the model-serving platform, **I want** to put our scoring API in front of the risk team's batch jobs, our customers on the internet and a data scientist's browser, each with exactly the reach they need, and to poke at a misbehaving replica from my laptop, **so that** nobody opens a LoadBalancer, a NodePort or a firewall rule to get their work done and I can answer "who can reach this, from where, and what does it cost" from one table.

**Context.** `churn-api` (US-04) runs in `ml-serving` on two EKS clusters, one in us-east-1 and one in eu-west-1; the EU one serves EU customers under a residency rule. The risk team scores loans nightly and wants to call the API from their batch jobs. Product wants a public endpoint by Friday. A data scientist wants their notebook in a browser without a tunnel per person. One replica returns 500s on one path and nobody can reproduce it through the load balancer. A bare-metal k3s site runs flannel. Today each of these is a ticket to the network team and a hand-written manifest.

## 2. Actors and identities

| Role | Who | Identity Convoy sees | Notes |
|---|---|---|---|
| Runs it | `sre-ana` | user `sre-ana`, groups `ml-serving` (primary), `sre` | owns `churn-api` through `ml-serving` |
| Calls it | `risk-raj`, risk team | user `risk-raj`, group `risk` | `batch --connect` from `/clusters/eks-us/risk` |
| Reaches a notebook | `ds-kim` | user `ds-kim`, group `ml-research` | `expose notebook --internal` |
| Approves | `sec-ola`, security | user `sec-ola`, group `security`, approver of the `expose-public` policy | answers the `--public` gate |
| Pays | `ml-serving` | the group's cost centre (ADR 0016 d8) | gateway share, data out, cross-zone traffic |
| Audits | security | unit receipts, grant events, the gateway's access logs | who could reach what, when |
| Operates the platform | platform team | internal and public Gateways, cert-manager with a private and a public issuer, external-dns, Cilium with FQDN policy on EKS, flannel on k3s, the inventory's delegated domains | |

## 3. Preconditions

- **Capabilities**: on both EKS clusters `gateway: internal, public (HTTP, gRPC)`, `netpol: enforced`, `admin-network-policy`, `fqdn-policy`, `certs: private, public`, `dns`, `flows: hubble`, `mcs`; `global-gateway` on neither. On `site-k3s`: `netpol: none`, `gateway: none`.
- **Packages**: the gateway package declares the auth policy for `--internal` (OIDC) and its mapping; cert-manager, external-dns, Cilium.
- **Mounts**: `/clusters/eks-us`, `/clusters/eks-eu` (residency `eu`), `/clusters/site-k3s`; the platform profile sets `egress=declared` on prod mounts.
- **Annotations**: `serve: port 8080/http, port 9090/http (metrics)`; `score-loans: egress: none, connects: churn-api`.
- **Platform defaults** `explain` must print: reach and span per address, the grant list, the gateway and issuer used, the price per gigabyte out and across zones, the enforcement mode, the approval policy for `--public`.

## 4. Main path

```bash
# Cluster reach, then a grant to one other team
[sre-ana] $ cd /clusters/eks-us/ml-serving
$ expose churn-api 8080
service churn-api.ml-serving.svc   reach cluster   allowed: ml-serving   enforced (BaselineAdminNetworkPolicy)
$ expose churn-api --allow=risk
grant: risk -> churn-api:8080   receipt event 4410
# the unit is managed by Argo CD (tracking label): Service, NetworkPolicy and grant written as intent, commit 91c4…   (L20.1)

[risk-raj] $ cd /clusters/eks-us/risk
$ explain batch --connect=/clusters/eks-us/ml-serving/churn-api -- score-loans
connects: churn-api.ml-serving.svc:8080  granted to risk (event 4410)  env CONVOY_ADDR_CHURN_API
path: same cluster; 3 zones; trafficDistribution PreferClose; cross-zone $0.01/GB each way, est. 40 GB/night   (L20.5)
egress: none declared beyond the connect; DNS, artifact store and IdP always reachable
[risk-raj] $ curl churn-api.ml-serving:8080/healthz
not reachable from a session: churn-api is granted to risk for workloads; no internal route exists   (L20.2)

# Public, gated
[sre-ana] $ expose churn-api --public
gate: expose-public (security, quorum 1, timeout 24 h)   pid 9120 state approval
  host churn-api.g-41c2.api.example.com   cert: public issuer   dns: external-dns   gateway: public-us
  price: gateway share $0.42/day, data out $0.09/GB
  TLS terminates at public-us (us-east-1); access logs to /s3/platform-logs-us                              (L20.8)
[sec-ola] $ approve 9120
[sre-ana] $ wait --approval 9120 && netstat
UNIT       REACH     SPAN    ADDRESS                               ALLOWED            ENDPOINTS  CERT
churn-api  public    eks-us  churn-api.g-41c2.api.example.com     *                  3/3        public, 89 d
churn-api  cluster   eks-us  churn-api.ml-serving.svc:8080        ml-serving, risk   3/3
?          route     eks-us  churn-api.g-41c2.api.example.com/v0  foreign HTTPRoute legacy-v0 (no convoy.io/group)  (L20.7)

# Both regions
$ expose churn-api --public --global
refused (exit 3): span needs global-gateway on eks-us and eks-eu; neither has it
  and: residency of /clusters/eks-eu is eu; TLS for a global route would terminate at public-us               (L20.8)

# A notebook in a browser
[ds-kim] $ run --unit notebook --gpu=1 jupyter && expose notebook --internal
route notebook.g-9d07.internal.example.com   login: OIDC (gateway package)   allowed people: ml-research
cert: private issuer (internal hostnames never go to public CT logs)                                         (L20.4)

# A broken replica, from the session and from the laptop
[sre-ana] $ curl -s churn-api:8080/score?id=77 -o /dev/null -w '%{http_code}\n'
port-forward under sre-ana to replica churn-api-2 (pod churn-api-6f9c-x2; 3 ready)   500
$ curl -s churn-api-0:8080/score?id=77 -o /dev/null -w '%{http_code}\n'   # one replica by name   (L20.2)
200
laptop$ convoy attach -L 9090:churn-api-2:9090      # metrics port, receipted per open
$ nohup ./loadtest.sh churn-api 500rps &
refused (exit 3): promoting to a Job loses the session's port-forward; the Job needs
  --connect=/clusters/eks-us/ml-serving/churn-api (granted: ml-serving, yes)   explain shows the reach     (L20.9)

# Revoking the grant
$ expose churn-api --allow=-risk
refused (exit 3): 1 running caller: risk/pid 9204 score-loans (connect, started 01:10); --yes revokes and tells risk   (L20.6)

# A canary with gRPC streams open
$ release churn-api sha256:5be1… --canary=10%
route weights 90/10 written as intent; weights apply to new connections only
  open long-lived connections on the old version: 412 (hubble)   drain: up to 10m (unit annotation)       (L20.10)

# The k3s site
$ caps /clusters/site-k3s | grep -E 'netpol|gateway'
netpol: none   gateway: none
$ expose /clusters/site-k3s/edge/collector --allow=sre
grant recorded, advisory: netpol none on site-k3s, reachability not enforced
$ expose /clusters/site-k3s/edge/collector --public
refused (exit 3): netpol none and no gateway on site-k3s

# Withdrawing the public address
$ expose churn-api --cluster
public route removed; host churn-api.g-41c2.api.example.com held for ml-serving for 30 days            (L20.3)
```

| Step | Convoy object | Kubernetes or cloud object | Where the state lives |
|---|---|---|---|
| `expose` (cluster) | an address, a grant list | Service, NetworkPolicy, BANP from the namespace template | API server, or the GitOps repository as intent |
| `--allow` | a receipted grant event | an ingress rule selecting the grantee's namespaces by `convoy.io/group` | unit receipt |
| `--connect` | env and an egress rule | caller NetworkPolicy; `trafficDistribution` on the Service | caller's receipt |
| `--public` | an approval object, an address | HTTPRoute on the public Gateway, Certificate, DNS record | `/sys/proc/<pid>/approval/<n>`, unit receipt |
| `--internal` | an address | HTTPRoute on the internal Gateway, the gateway's auth policy, a private Certificate | unit receipt |
| session `curl` | hosts entry | `pods/portforward` under the user's token | the session |
| `attach -L` | a forward, receipted | the same port-forward through the attach channel | the session, receipt |
| `netstat` | the table | Services, routes, EndpointSlices, Certificates, flows | `/sys/unit/<name>/net/` |
| `release --canary` | weights, drain | HTTPRoute weights as intent | repository, receipt |

## 5. Abstraction coverage

| # | Aspect | Touched | How this story uses it | What to verify |
|---|---|---|---|---|
| A1 | Pipe | ○ | the risk batch still writes artifacts; only the call to the API is a socket | |
| A2 | Command | ○ | annotations declare ports, egress and connects | |
| A3 | Session | ● | the hosts view, `attach -L`, promotion by `nohup` (L20.2, L20.9) | AC-20.3, AC-20.6 |
| A4 | Backend | – | | |
| A5 | Contract | – | | |
| A6 | Device tree | ● | `caps` keys for gateways, netpol, issuers, span; `/sys/unit/<name>/net/` | AC-20.7 |
| A7 | Signals | – | | |
| A8 | Secrets | – | | |
| A9 | Cost | ● | gateway share, data out, cross-zone and cross-region per gigabyte (L20.5) | AC-20.4 |
| A10 | Identity | ● | grants by group, the `--public` gate, the `--internal` login, session reach by token (L20.2) | AC-20.1, AC-20.3 |
| A11 | Defaults | ● | default-deny and the group boundary from the template; issuers per gateway; drain default | AC-20.7 |
| A12 | Storage | – | | |
| A13 | Networking | ● | every decision of ADR 0019 | all |
| A14 | Time | ○ | hostname hold, drain window (L20.3, L20.10) | AC-20.5 |
| A15 | Audit | ● | grant and revocation events, approval in the receipt, access-log location (L20.6, L20.8) | AC-20.2 |
| A16 | Durability | ○ | a held hostname outlives the route (L20.3) | |

## 6. Loophole probes

| # | Probe | Question for this story | Design answer | Finding |
|---|---|---|---|---|
| P1 | Imperative vs declarative | `churn-api` is Argo-managed; `expose` writes a Service, routes and a NetworkPolicy, and `--allow` edits it | ADR 0019 does not say; ADR 0010 amendment 1 says every mutation of a managed object is intent | **L20.1** |
| P2 | POSIX cannot hold | `curl` from the session reaches one pod over a port-forward, not the Service; a member of a granted group has no `pods/portforward` in the owner's namespace | the hosts view says which pod; a granted, non-owning group reaches through an internal route or not at all | **L20.2** |
| P3 | Orphaned state | `expose --cluster` drops the public route; the hostname, its DNS record and certificate are released and another group can claim the name the internet still calls | a withdrawn hostname stays with its group for a hold period | **L20.3** |
| P4 | Identity leak | a public CA logs every hostname it certifies in Certificate Transparency; internal unit names and group slugs would be world-readable | internal routes take certificates from a private issuer only | **L20.4** |
| P5 | Invisible cost | the risk batch calls the API across three zones nightly; ADR 0019 prices data out to the internet, not cross-zone or cross-region traffic between units | `explain --connect` prints the path class and its price; Services the shell writes prefer the same zone | **L20.5** |
| P6 | Lying cache | `CONVOY_ADDR_CHURN_API` is injected at plan time | it is a Service name, stable across rollouts and re-releases in place | ok |
| P7 | Silent failure | revoking a grant turns a running caller's requests into timeouts, since NetworkPolicy drops silently | the revoke lists running callers, needs `--yes` and tells their group | **L20.6** |
| P8 | Partial fleet | `--global` over two clusters, neither with a span capability; flannel on k3s | refused, naming the capability per cluster; on k3s the grant is advisory and `--public` refused (decision 6) | ok |
| P9 | Created elsewhere | a foreign HTTPRoute on the same Gateway claims a path under the unit's hostname; Gateway API resolves the conflict by age, silently | `expose` pre-flights hostname and path against every route on the Gateway and names the owner or `?` | **L20.7** |
| P10 | Multi-party | the approver of `--public` needs to see who will reach what | the plan's shape carries host, allowed set, gateway and price (ADR 0017 decision 6) | ok |
| P11 | Data at rest | TLS terminates at the gateway and the gateway writes access logs with URLs and query strings; for a global route both may sit outside the unit's residency | ADR 0019 checks residency per cluster spanned, not where TLS ends or logs land | **L20.8** |
| P12 | Inheritance | a session command reaches the API through the hosts view; `nohup` promotes it to a Job that has no port-forward and no `--connect` | promotion re-plans reach and refuses naming the `--connect` it needs | **L20.9** |

A thirteenth question, from the ADR's own open list: `release --canary` moves weights, but HTTP/2 and gRPC streams, WebSockets and keep-alive connections stay on the replica they opened on, so a 10 % canary can take hours to see traffic and a rollback leaves the bad version serving open streams. **L20.10**. A fourteenth: a notebook exposed `--internal` is reachable by every member of its group, while `ds-kim` thinks of it as theirs. **L20.11**, accepted.

## 7. Failure matrix

| Failure | Detected by | Exit | Receipt says | Recovery verb |
|---|---|---|---|---|
| caller's group not granted | `explain`, plan pre-flight | 3 | target group, `expose --allow` spelling | owner runs `expose --allow` |
| `--public` not approved in time | the gate's timeout (L18.6) | 3 | expired | re-request |
| no gateway or span capability | pre-flight | 3 | the capability, per cluster | platform installs, or narrower reach |
| `netpol: none` | `caps`, `explain` | ok, marked | advisory grants, `reachability not enforced` | platform changes CNI |
| hostname or path conflict on the Gateway | pre-flight (L20.7) | 3 | the other route and its owner | `--host`, or the owner withdraws |
| grant revoked with running callers | pre-flight (L20.6) | 3 without `--yes` | the callers | `--yes`, after telling them |
| TLS termination or logs outside residency | pre-flight (L20.8) | 3 | where each lands | a gateway in residency |
| promotion loses session reach | `nohup` pre-flight (L20.9) | 3 | the `--connect` needed | `nohup` with `--connect` |
| old version still serving streams after release | `release` (L20.10) | ok, reported | open connections, drain | `release --drain=<d>` or wait |

## 8. Acceptance criteria

- **AC-20.1** Given `expose <unit> --allow=<group>` and a caller in that group with `--connect`, then the call succeeds from a workload, the grant is a receipted event, and a caller from any other group is refused at plan time naming the target's group.
- **AC-20.2** Given `expose --allow=-<group>` while a process of that group is connected, then the verb refuses without `--yes`, lists the callers, and on `--yes` the callers' group is told and their `netstat` shows `grant revoked`.
- **AC-20.3** Given a session command that names a unit or one of its replicas (`<unit>-<n>`), then it reaches that pod, or one ready pod, by port-forward under the user's token when the user's group owns the unit, the unit's internal route when the user's group is only granted, and nothing otherwise, and the session says which.
- **AC-20.4** Given `explain` on a plan with `--connect` or `--public`, then it prints the path class (same zone, cross-zone, cross-region, internet) and the price per gigabyte of each.
- **AC-20.5** Given a public or internal route withdrawn, then the hostname stays reserved to its group for the hold period and a claim by another group is refused naming the date it frees.
- **AC-20.6** Given `nohup` on a session command that reached a unit through the hosts view, then promotion refuses unless the Job carries a granted `--connect`, and says so.
- **AC-20.7** Given `--public` or `--global` on a residency-bound unit, then the plan lists where TLS terminates and where access logs land, and refuses when either is outside the residency.
- **AC-20.8** Given `release --canary` on a unit with long-lived connections, then the receipt and `release` output state that weights apply to new connections, report open connections on the old version, and the drain window used.

## 9. Persona lenses

- **SRE**: wants one table for reach and endpoints, a way to hit one replica from the laptop without a ticket, and a release that tells them the old version is still serving streams.
- **DO**: wants every Service, route, grant and weight on a managed unit written as intent, so Argo never reverts them.
- **DS**: wants a browser URL for a notebook with their login, no tunnel, and to know who else can open it.
- **SEC**: wants grants by group with a receipt, revocations that cannot be silent, internal names kept out of CT logs, TLS and logs inside residency, and no LoadBalancer anyone can create.
- **PE**: wants the boundary in the template and the gateway in their hands, `caps` to say what each cluster enforces, and hostnames that never change hands quietly.
- **FIN**: wants cross-zone and data-out traffic in the band and in the receipt, not discovered on the bill.

## 10. Findings

| Id | Probe | Finding | Proposed disposition | Touches |
|---|---|---|---|---|
| L20.1 | P1 | ADR 0019 lists what `expose` writes but not how on a GitOps-managed unit; a Service, route or NetworkPolicy written directly would be reverted by the reconciler, and a grant would vanish. | **rule** (applied: ADR 0019 rules; command map; roadmap M11): on a managed unit `expose`, `--allow`, `--off` and reach changes write the Service, routes and NetworkPolicy as intent (ADR 0010 amendment 1) and wait for reconciliation; the receipt carries the commit; `explain` prints the repository | ADR 0019 d2, d3; ADR 0010 a1; S4 |
| L20.2 | P2 | Decision 7 forwards to any unit in a mounted namespace under the user's token, but a member of a granted group has no `pods/portforward` in the owner's namespace, so the hosts view promises a reach RBAC refuses; and granting it would open every port of every pod there. | **amend** (applied: ADR 0019 d7 revised) ADR 0019 decision 7: the hosts view maps units the user's groups own to a port-forward, and each ready replica of them as `<unit>-<n>` so one replica can be addressed; units they are only granted to the unit's `--internal` route when one exists, and nothing else; the session says `not reachable from a session` with the reason; a grant never gives `pods/portforward` | ADR 0019 d3, d7; ADR 0016 d8 |
| L20.3 | P3 | Withdrawing a route releases its hostname; another group can claim it while clients, bookmarks and webhooks still call it, which is a takeover by the book. | **rule** (applied: ADR 0019 rules; command map; roadmap M8): a hostname stays reserved to its group for a hold period after its last route is removed (30 days unless the platform sets it); a claim by another group is refused naming the date; `netstat --held` lists them | ADR 0019 d8; ADR 0018 d7 |
| L20.4 | P4 | A public CA records every certificate in Certificate Transparency logs, so internal hostnames with unit names and group slugs would be world-readable. | **rule** (applied: ADR 0019 rules; command map; roadmap M9): `--internal` routes take certificates only from an issuer `caps` lists as private; a gateway without one refuses `--internal` with a public issuer unless the platform sets `internal-ct: allowed`, and `explain` prints the issuer | ADR 0019 d2, d8 |
| L20.5 | P5 | Decision 10 prices data out to the internet, but a caller across zones or regions pays per gigabyte on every call, and nothing shows it. | **rule** (applied: ADR 0019 rules; command map; roadmap M11): `explain` on `--connect` prints the path class (same zone, cross-zone, cross-region) with the price per gigabyte from `/sys/k8s/<id>/link` and an estimate from the unit's history; Services the shell writes set `trafficDistribution: PreferClose` unless the unit opts out; the receipt settles cross-zone bytes under `caps: flows` | ADR 0019 d10; ADR 0011 rule L10.5 |
| L20.6 | P7 | A revoked grant turns a running caller's requests into timeouts; NetworkPolicy drops, it does not refuse, and the caller's team learns from an incident. | **rule** (applied: ADR 0019 rules; command map; roadmap M11): `expose --allow=-<group>` lists running callers from the connect references, refuses without `--yes` when any exist, and on `--yes` notifies the callers' group by its page route; their `netstat` and `ps` show `grant revoked` | ADR 0019 d3; ADR 0017 (page route) |
| L20.7 | P9 | A route written outside the shell can claim a hostname or path on the same Gateway; the Gateway API resolves the overlap by age without telling either side. | **rule** (applied: ADR 0019 rules; command map; roadmap M11): `expose` pre-flights hostname and path against every route attached to the target Gateway, refuses on overlap naming the owner or `?`, and `netstat` lists foreign routes to a unit's Service or hostname as `?` | ADR 0019 d5, d9; ADR 0018 d5; S9 |
| L20.8 | P11 | Residency is checked per cluster spanned, but TLS ends at the gateway and access logs with URLs and query strings land wherever the gateway writes them; a global or edge gateway may do both outside the residency. | **amend** (applied: ADR 0019 d2 revised) ADR 0019 decision 2: TLS termination and access-log location are residency facts; `caps` lists both per gateway; `--internal`, `--public` and `--global` on a residency-bound unit refuse a gateway whose termination or logs lie outside it; `explain` prints both | ADR 0019 d2, d5; ADR 0011 a3; S13 |
| L20.9 | P12 | A session command reaches units through the hosts view; promoting it to a Job (`nohup`, decision 7) leaves it with no port-forward and no `--connect`, so the Job fails at its first call. | **rule** (applied: ADR 0019 rules; ADR 0005 rules; command map; roadmap M11): promotion re-plans reach: every unit the command named through the hosts view must be a granted `--connect` on the Job, else promotion refuses naming them; `explain` prints the result | ADR 0019 d3, d7; ADR 0005 |
| L20.10 | none | Route weights apply to new connections only; gRPC streams, WebSockets and keep-alive connections stay on the old version, so a canary sees less traffic than its weight and a rollback leaves the bad version serving. | **rule** (applied: ADR 0019 rules; ADR 0013 rules; command map; roadmap M11): `release` and `expose --off` report open long-lived connections on the old version where `caps: flows` exists (`unknown` otherwise), apply a drain window (`--drain=<d>`, default from the unit annotation) after which the old replicas close connections, and the receipt records both | ADR 0019 d9; ADR 0013 (`release`) |
| L20.11 | none | An `--internal` notebook is reachable by every member of its allowed groups; its author thinks of it as personal. | **accepted**: reach goes to groups (ADR 0016); a person who wants a private notebook uses a group of one; the gateway forwards the user so the notebook can enforce its own login; `explain` prints the allowed groups and their size | ADR 0019 d2, d3; ADR 0016 d3 |
