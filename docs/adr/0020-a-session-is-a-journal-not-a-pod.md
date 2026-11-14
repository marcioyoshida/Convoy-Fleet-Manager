# ADR 0020: A session is a journal and a login, not a pod to keep alive

**Status:** Draft · 2026-11-10 · revised 2026-11-10 with user-story findings L21.2 and L21.4 (marked inline)
**Supersedes / relates to:** ADR 0005 (the session is a server-side object; this record is its design); docs/stories.md (session placement and durability, the architecture-changing row); docs/debate-review.md §7 and ADR 0009 (the session watches each object and persists its exit code, which this record moves to the controller; the receipt spool of ADR 0009 amendment 1 is unchanged); ADR 0006 (detach is not destroy); ADR 0011 (the `self` device, amendment 4 on disconnected devices); ADR 0014 (completion state per session); ADR 0016 decisions 1, 5, 9, 10 and 11 (the login token, launched processes as `svc:<group>`, lifecycle at the provider, mount profiles, the session's location as a residency fact); ADR 0017 decision 4 (the controller owns the gate); ADR 0019 decision 7 (the hosts view) and rule L20.9 (promotion re-plans reach); rule L6.7 (prompts in non-interactive sessions)

## Context

ADR 0005 put the session on the server so a sleeping laptop kills nothing and the audit stream is trustworthy. It did not say where the pod runs, what happens when the pod dies, how a terminal attaches, what two terminals on one session see, or when a session ends. Every other record leans on it: mounts and env live in it, the hosts view of ADR 0019 runs in it, completion counters of ADR 0014 are per session, and docs/debate-review.md makes it the component that watches Jobs and writes their exit codes. The user stories never reached the attach protocol, and the owner asked how contexts are added and switched and what the prompt shows.

Facts that constrain the design:

- A pod is mortal. Node drain, eviction, preemption, a spot reclaim, an OOM or a cluster upgrade ends it, and its replacement is a new pod with empty memory. Anything held only in a pod's memory is lost with it.
- Since ADR 0016 decision 5 every launched process runs as its group's service identity and never needs the person's token, and since ADR 0017 decision 4 the controller owns gates and resumes stages. Nothing a running process needs is required to live in the session any more, except the receipt writer docs/debate-review.md put there.
- The login token is short-lived. Refreshing it needs a refresh token, which is a long-lived credential; wherever it is stored is where it can be stolen.
- Users run arbitrary toolbox commands in the session's local backend; the pod is multi-tenant infrastructure running untrusted code.
- A session's state is small: a mount table, an env table, shell variables and functions, a cwd per shell, a jobs table, hosts-view forwards, completion counters. History is the audit stream, which already lives elsewhere.
- A user at an edge site with the WAN down cannot reach a central session, and a central session cannot reach the site (ADR 0011 amendment 4).
- A kubeconfig context bundles a cluster, a credential and a namespace. In this design those are a device (ADR 0011), the session's identity (ADR 0016) and a directory; there is nothing left for a context to hold.

## Decision

1. **Nothing a launched process needs lives only in the session.** Receipts of remote processes, exit codes, gates, holds, cron entries and units are written by the Convoy controller, which watches the objects the planner wrote; the session is a client of the controller for everything that outlives a command. Losing the session loses no remote process, no receipt and no approval. This moves the receipt writer of ADR 0009 and docs/debate-review.md §7 from the session to the controller; where the controller cannot reach a device, the per-device spool of ADR 0009 amendment 1 still persists the receipt where the object completed.

2. **The session's state is a journal, and the pod is a cache of it** (revised 2026-11-10, L21.2). Every change to session state (a mount, an export, a function, a `cd`, a new shell, a job started) is appended to a journal before it takes effect, in the artifact store of the session's residency (ADR 0016 decision 11) under the retention class `session`. The journal never holds a token, a secret value or command output; env entries that are secret references stay references (ADR 0007). The shell cannot tell a secret from a string, so the journal is encrypted under the owning group's key in the residency's KMS, and an exported value the audit redactor flags is kept in memory only: the shell says so and suggests `secret set`, and after a restore such variables are unset and listed by name. A session pod is restored from the journal on any session host of the same residency, and `/sys/session/<id>/journal` shows its last entry and size.

3. **Commands are recorded before they run; local commands do not survive the pod.** The audit stream records each command line before execution (ADR 0005), so a command that was running when the pod died is still in `history`. A local command in flight is lost with the pod: its receipt is written by the restored session with exit 137 and reason `session-lost`, and `jobs` lists it with the `replay` spelling when its annotation says deterministic. Nothing is retried on its own, because the side effects are unknown. The planner keeps a stage local only when it is interactive or its estimate is under the platform's `local_max` (ADR 0010; `explain` prints it); longer work is a Job, and `nohup` promotes explicitly (rule L20.9).

4. **The terminal refreshes the token; the pod never holds a refresh token.** The thin client logs in at the identity provider and pushes each new access token over the attach channel; the session holds the access token in memory only. A session with no attached client keeps running its local commands until the token expires, then becomes `dormant`: its own calls stop, its remote processes go on as their service identities, and the next attach with a token for the same user wakes it. A user removed at the provider cannot obtain one, which is how ADR 0016 decision 9 ends their sessions. `id` prints the token's expiry; `/sys/session/<id>/state` reads `active`, `detached`, `dormant` or `restoring`.

5. **One attach protocol, through the platform's gateway.** The client reaches the session through an `--internal` route on the platform's gateway (ADR 0019) with the login of ADR 0016, over one WebSocket with channels for terminal input and output, resize, token refresh, port forwards (`attach -L`, ADR 0019 decision 7) and control. The client never reaches the API server for the session and needs no kubeconfig. Reconnect resumes the same terminal at the last acknowledged output offset, so a dropped connection loses no output and repeats none.

6. **A session is a login; each terminal is a shell** (revised 2026-11-10, L21.4). The session holds what a Unix login shares: the mount table, the hosts view, the jobs of every shell, pending approvals, completion counters, the token. Each attached terminal runs its own shell on its own `/dev/pts/<n>`, with its own cwd, shell variables, functions and foreground job; `export` is per shell and inherited at launch, as in POSIX. `attach` opens a new shell; `attach --tty=<n>` reattaches to a shell left running; `attach --tty=<n> --share` mirrors one shell in a second terminal of the same user, receipted; another user never gets input to a shell, because the shell holds its user's token. A member of one of the owner's groups may `attach <user> --tty=<n> --watch`, read-only, after the owner accepts in that shell; every watch is receipted and listed by `who`. A `mount` or `umount` is session-wide and printed on every other attached terminal. `tty` prints the terminal, `who` lists the user's sessions and terminals with host and state (root sees all), and `exit` ends a shell.

7. **The session ends by `logout`, by the last shell exiting, or by idle; it never ends a remote process.** `logout` with local commands still running refuses without `--yes`, listing them with the `nohup` spelling (rule L6.7). With no attached client and no local command for the platform's `session_idle` (8 h unless set), the session is checkpointed and its pod deleted; the next `attach` restores it from the journal and shows the same shells, cwd and jobs. A session whose journal has had no attach for the platform's `session_expire` (30 days unless set) is ended and its journal kept for the audit retention. The session pod's cost is charged to the user's primary group and shown by `cost --session`.

8. **Session hosts are designated by the platform, per residency.** The inventory marks clusters as session hosts (`caps: session-host`) per residency; a session runs on a host in the residency of the user's primary group, and a user whose groups span residencies has one session per residency (ADR 0016 decision 11), chosen at `attach` with `--residency` and named by `id`. The host is the `self` device of ADR 0011. A host that is lost or drained moves its sessions to another host of the same residency by restoring their journals; a residency with one host has sessions that wait in `restoring` until it returns. Session pods run under the platform's sandbox runtime class where `caps: sandbox` exists, in a namespace per host that admits nothing but session pods, with no ServiceAccount rights beyond reaching the controller.

9. **An edge site can host its own sessions.** A site with `caps: session-host` serves sessions for users at the site; there, the site's devices are ready and the rest of the fleet is `disconnected` by ADR 0011 amendment 4 while the WAN is down. A user may hold a central session and a site session; they share the profile, the objects and the controller's state where reachable, and nothing else, and `who` lists both. Its journal is written to the site's artifact store and copied with the site's other spools when the link returns.

10. **There is no context object: a mount adds, `cd` switches.** A kubeconfig context is a device, an identity and a namespace, and each already has a home. Adding a cluster is `mount [-o identity=…] /dev/k8s/by-id/<id> /clusters/<name>`, persistent for a group through `/etc/fstab.d/<group>` (ADR 0016 decision 10) and for the user through their own profile; switching is `cd /clusters/<name>/<namespace>`, with `cd -`, `pushd` and `popd` as in POSIX; one command elsewhere is an absolute path or `( cd … && cmd )`. A kubeconfig is never imported: a cluster that is not in the inventory is not a device, and a credential in a file is the shared secret ADR 0016 forbids. `kubectl` inside the session gets a generated kubeconfig for the cwd's mount, with the session's exchanged identity and the cwd's namespace, rewritten on every `cd`.

11. **The prompt shows where a command will land and with what identity, from session state only.** `PS1` keeps the POSIX escapes and gains:

    | Escape | Shows | Source |
    |---|---|---|
    | `\u` | the user | the token |
    | `\g` | the primary group (`newgrp` changes it) | the token |
    | `\h` | the session host (`self`) and, when more than one, the residency | the session |
    | `\w` | the cwd path, `~` for home | the shell |
    | `\K` | the cwd's mount name (`eks-us`) | the mount table |
    | `\N` | the cwd's namespace | the mount table, `nsmap` |
    | `\I` | the identity mark when it is not `user`: `assumed role/deployer`, `svc:payments`, `shared`, `anonymous` | the mount table |
    | `\m` | marks on the cwd's device: `disconnected 4m`, `admission:shell-only`, `netpol:none`, `ro` | cached device state |
    | `\a` | pending approvals the user is waiting on, and those waiting on the user's groups | the controller, cached |
    | `\$` | `#` under `svc:root` (`sudo -i`), `$` otherwise | the session |

    A device may carry a platform hint (`/sys/k8s/<id>/prompt`, for example `colour=red` on production), which the default prompt uses. The platform profile sets the default `PS1`, for example `\u@\g \K:\N \I\m\a\$ `, which renders as `ana@ml-serving eks-us:payments (assumed role/deployer) [disconnected 4m] (2 pending) $ `. Every escape reads session state or device state cached from hotplug events (ADR 0011); computing the prompt never calls an API, so it costs nothing and cannot hang. Non-interactive sessions never render it.

12. **The session is upgraded only at a boundary.** A new session image is applied when a session is restored (after idle, a host loss or a reattach); a running session is never replaced under a user. A security upgrade the platform must force is a drain: every attached terminal is told the deadline, local commands get the platform's grace to finish, then the session is checkpointed and restored on the new image, and the receipt of each lost local command says `session-upgrade`.

## Declined

| Proposal | Why not | What replaces it |
|---|---|---|
| a StatefulSet with a PVC per user for session state | binds the session to one zone and one cluster; a host loss strands the volume | the journal in the residency's artifact store (decision 2) |
| the session pod holding a refresh token | a long-lived credential at rest in multi-tenant infrastructure | the client refreshes and pushes access tokens (decision 4) |
| retrying lost local commands on restore | side effects unknown; a retry may write twice | receipt `session-lost`, `replay` where deterministic (decision 3) |
| attaching through the API server (`kubectl attach`) | needs a kubeconfig and RBAC on the session namespace for every user, and gives no token channel | one WebSocket through the platform's gateway (decision 5) |
| SSH as the attach protocol | key distribution is a second identity system | the WebSocket protocol; an SSH bridge, if wanted, is a package that authenticates through the same login |
| one shell per session (tmux inside the pod) | two terminals would fight over one cwd and one foreground job | a login with a shell per terminal (decision 6) |
| contexts as a named object (`context add`, `context use`) | duplicates the device, the identity and the directory | `mount` and `cd` (decision 10) |
| importing a user's kubeconfig | bypasses the inventory and carries a static credential | the inventory's devices; a generated kubeconfig for `kubectl` |
| a prompt that queries cluster state | one API call per prompt is latency, cost and a hang when a device is down | escapes over cached state (decision 11) |
| the user choosing the session host | a session in the wrong place is data at rest in the wrong place (S13) | hosts per residency (decision 8) |

## Consequences

- ADR 0005's consequence "this decision precedes" identity, ownership and networking is discharged: all three are drafted, and the session they store state in now has a design.
- The controller grows: it writes receipts for remote processes, which the session read from it before; M2 carries the controller's receipt writer, the journal and its restore, the attach protocol and the prompt escapes, and session hosts per residency move into M8 with the inventory.
- New `caps` keys: `session-host`, `sandbox`. New platform defaults: `local_max`, `session_idle`, `session_expire`, the default `PS1`, the forced-upgrade grace. New paths: `/sys/session/<id>/` with `state`, `host`, `journal`, `ttys`, `token-expiry`, `version` and `cost`, beside the completion state of ADR 0014.
- New or specified verbs and flags: `attach [--tty=<n>] [--share] [--residency=<r>] [-L …]`, `tty`, `who`, `logout [--yes]`, `cost --session`; `newgrp` already exists in ADR 0016.
- A lost session pod costs the user the local commands in flight and a few seconds of restore; it costs nothing else. The trade is that every state change pays one journal append.
- Settled 2026-11-10: a shell is shared only with its own user; another member may watch, read-only and with consent (L21.4, decision 6).
- Open, to be settled against this ADR: the journal's write path at sites with intermittent object storage; the restore latency target per residency; session limits per user and per group; how `history` across a user's several sessions is merged.
- [US-21](../user-stories/US-21-session-survives.md) probes this record under the Data Engineer, SRE and Security personas with a context added and switched, a session host drained under a local sort, a second terminal and a colleague's watch, an overnight token expiry with a follow running, a secret typed into `export`, `newgrp` across residencies and an edge site without its link. It raised eleven findings; two were applied inline the same day: L21.2 (the journal is encrypted and a value the redactor flags is never journaled; decision 2) and L21.4 (`--share` is the same user only, a group member may watch read-only with consent, which settles an open item; decision 6). Eight are rules, written below, in ADR 0016, in ROADMAP M2, M7 and M8 and in the command map: every eviction of a session pod handled as a drain (L21.1), no token in the generated kubeconfig (L21.3), `newgrp` across residencies refused (L21.5), bounded scrollback (L21.6), the age of prompt marks (L21.7), local commands stopped at dormancy (L21.8), no exec into session pods without break-glass (L21.9) and a warning past `local_max` (L21.10). L21.11 accepts that a residency with one session host waits.

Note · 2026-11-11: the draft [ADR 0022](0022-the-laptop-lends-a-tree-the-cluster-builds-it.md) adds a lend channel to the attach protocol of decision 5: the client sends content-addressed snapshots of a declared directory, by request from its manifest, and `attach` gains `--lend` and `--env-of`.

## Rules · 2026-11-10 · from the user-story probes

Each line is a `rule` finding from `docs/user-stories/README.md` written here as decided; the story holds the reasoning.

- **L21.1** (US-21) session pods carry a PodDisruptionBudget and `karpenter.sh/do-not-disrupt` while a local command runs, run only on capacity the session host marks non-spot, and any eviction (a node drain, a consolidation, a reclaim) is handled as the drain of decision 12: terminals told the deadline and the commands at risk, the platform's grace, then checkpoint and restore.
- **L21.3** (US-21) the generated kubeconfig of decision 10 uses an exec credential that asks the session over a local socket for a token per call, scoped to the cwd's mount; no token is ever written to the file.
- **L21.5** (US-21) `newgrp` to a group of another residency is refused in the session, naming the residency of each and the `attach --residency` spelling (decision 8, ADR 0016 decision 11).
- **L21.6** (US-21) each terminal keeps a bounded scrollback in memory (platform default); a resume beyond it says how many bytes were dropped; after a restore the shell says that scrollback before the restore time was not kept; output meant to last is redirected to a path.
- **L21.7** (US-21) each cached device state carries its age; prompt marks older than the platform's `prompt_stale` render as `?`, and `mount` shows the age.
- **L21.8** (US-21) at dormancy local commands are stopped (SIGSTOP), listed by `who` and resumed at the next attach; after `session_idle` dormant they are ended with receipt `session-dormant` and the pod is checkpointed.
- **L21.9** (US-21) `pods/exec`, `pods/attach`, `pods/portforward` and ephemeral containers in session namespaces are granted to no one; break-glass is `sudo` by `wheel` with a second member's approval (ADR 0016 decision 6), the session is told, and `who` shows `inspected by` with the approval.
- **L21.10** (US-21) when a local command passes `local_max` the shell warns once in its terminal with the `nohup` spelling for next time, the receipt records `over local_max`, and the planner's estimate for that command class learns from it.
- **L26.14** (US-26) the redactor of rule L21.2 also covers what a telemetry read prints, before the terminal and the journal (full rule in ADR 0023).
- **L31.3** (US-31) `write` and `wall` to a user with no attached terminal go to the session journal and are shown at the next attach (full rule in [ADR 0027](0027-convoy-tells-a-group-about-its-own-objects.md)).
