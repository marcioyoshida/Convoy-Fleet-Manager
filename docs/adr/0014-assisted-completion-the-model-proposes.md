# ADR 0014: Completion is type-directed; a model may rank and translate, and the model is a device

**Status:** Draft · 2026-11-09 · revised 2026-11-09 with user-story findings L14.2 and L14.4 (marked inline)
**Supersedes / relates to:** ADR 0002 (annotation classes give the grammar its types); ADR 0004 (contract flags); ADR 0005 (the session owns the context completion reads); ADR 0007 (secrets never reach the client or a model); ADR 0011 (device classes, mount options, `/sys`, `disconnected`); ADR 0013 (the verb set, `explain --json`, uniform refusals); docs/user-stories/README.md probes P4 (identity leak), P5 (invisible cost), P8 (partial fleet), P11 (data at rest)

Note · 2026-11-10: the draft [ADR 0021](0021-a-node-is-tested-before-it-is-trusted.md) decision 6 is the one place a model's judgment leads to an action with no cursor: the node health agent, a session under a platform service identity, returns typed verdicts that cite probe receipts, from a fixed action set, within a disruption budget. Decision 9 below is unchanged for every session a person or a team owns.

## Context

The owner's remark: with language models attached to the shell, the next word can always be pulled from an options list with a tab, which gives the language semantic support.

What the design already gives completion, before any model is involved:

- The next token is almost always drawn from a closed set the session owns: devices under `/dev/<class>/by-id` and their views, mount points and their options, pids and units, annotation classes, the verbs of ADR 0013 and the flags bound to them, platform defaults, capabilities.
- The pipe is typed. The annotation database (ADR 0002) says what each command accepts and emits, so after `sort |` the candidates are verbs whose input class accepts a sorted stream, and `uniq -c` ranks above `tail -f`. That is type-directed hole filling, not prefix matching.
- `explain` (ADR 0013 decision 3) prints, without executing, what a line would do, including its cost band and its refusals. A completion can show the consequence of a candidate, not only its spelling.

What a model adds is narrower than it looks: ranking inside a candidate set the shell computed, a one-line gloss per candidate, and translation of a natural-language comment into a draft line. What a model must not add is a token the grammar does not allow, a name the user may not see, or a call the user did not ask to pay for.

The probes already written constrain the build: identity (P4), cost (P5), disconnected sites (P8, US-10), data at rest and residency (P11, US-08), and the rule from ADR 0007 that no secret value ever reaches the client.

## Decision

1. **Completion is grammar- and type-directed first, and needs no model.** The session computes the candidate set from its own state for every cursor position:

   | Position | Candidate set | Source |
   |---|---|---|
   | command | verbs (ADR 0013), toolbox commands, package executables, builtins | annotation database, `pkg list`, `tool list` |
   | after `\|` | commands whose input class accepts the upstream output class; a follow upstream excludes aggregate and fan-in (L12.2) | annotation database |
   | flag | the flags bound to that verb in ADR 0013, with the contract field each sets | ADR 0013 tables |
   | path | entries under the mount, from the metadata cache, with age | mount table, `/sys`, fileserver |
   | device | `/dev/<class>/by-id` and views, with state | inventory (ADR 0011) |
   | mount option | the options the device's driver knows | driver `caps` |
   | pid, unit, group | the session's and the namespace's processes and units | `ps` index |
   | `--engine`, `--on-budget`, `--deadline-from`, class names | fixed enumerations | ADR 0003, 0004 |

   The set is filtered by the session identity before anything else sees it (decision 4). A candidate that is not in the set cannot be completed, by a model or otherwise.

2. **The model proposes, the planner disposes.** A model attached to the session may do three things and nothing else:
   - **rank** the candidate set of decision 1 and attach a one-line gloss to each candidate, using what `explain --json` would print for the line so far;
   - **translate** a comment (`# reprocess yesterday's orders into eu`) into a draft line, inserted at the prompt, never run; the draft is ordinary input and passes through `explain` like anything typed;
   - **gloss** a refusal or a warning from `explain` with a suggested alternative, which is again only input.

   A proposal that contains a token outside the candidate set, or that does not type-check, is dropped by the session and counted (decision 8). The model never completes a value for a secret, a token, an approval or a confirmation, and `--with-data`, `--force` and `rmvol` are never pre-filled.

3. **The model is a device.** It lives under `/dev/model/by-id/<id>` and follows every rule of ADR 0011: views by provider and topology, mounted with options, opened under an identity, metadata and price under `/sys/model/<id>/`, hotplug from the inventory. A model served inside a cluster on `/dev/accel` and a hosted model are the same class with different `/sys` files. Mount options: `identity`, `residency` (the context never leaves the region the option names), `retention=<duration>` (the provider-side retention the mount accepts; `/sys/model/<id>/retention` states what the provider keeps and for how long, from the inventory, and a regulated profile requires `retention=0` and refuses the mount otherwise; the mount's audit record carries the value, L14.4), `scope=rank\|translate\|gloss` (which of decision 2's three the mount allows), `cap=<amount per day>`, `ro` is implied. The platform default `completion_device` names the device a new session uses; `completion=off` on a session or a mount disables it; there is no API key in a shell setting and never a value in the environment (ADR 0007).

   ```
   mount -o identity=secret:completion-reader,residency=eu,scope=rank,translate,cap=2.00 \
         /dev/model/by-id/bedrock-eu-west-1-claude… /dev/model/self
   ```

4. **Identity filters before the model sees anything.** The candidate set is computed under the session identity and the mount's identity for each path: a namespace, topic, secret name or pid the identity cannot list is not a candidate and is not in the context sent to the model. The model does not get a wider view than the user, so completion cannot become the identity leak probe in a new coat (P4).

5. **The context is scrubbed and residency-bound.** What goes to the model is the line so far, the candidate set with its glosses, and the `explain --json` of the line so far. It never contains a secret value (none exists in the session, ADR 0007), the contents of any artifact, or the receipt bodies of other users. Secret names and topic names are data at rest about the tenant (P11), so the `residency` option on the model mount decides whether they may be sent at all; a mount without `residency` on a platform that requires it is refused at mount time. The comment a user types for translation is free text and may contain a password, a token or a customer identifier, so every outbound context passes the same scrub that `convoy-put` applies to artifacts (ADR 0007 amendment 2): a match refuses the translation, names the rule without quoting the match, and increments the `scrubbed` counter; the rule set is a platform default and `explain` prints its version (L14.2).

6. **Every call is a ledger line.** Each ranking or translation is an inference with a price from `/sys/model/<id>/price`, charged to the session under `/sys/cost/session/completion`, visible in `time` for the session and in the FinOps view like any process. The `cap` option is a daily budget per mount; at the cap, completion falls back to decision 1 with a notice, never to a silent stop. Prefix completions that resolve to one candidate never call the model.

7. **Completion never blocks on a device.** Candidate sets for paths come from the metadata cache with their age shown; a device in `disconnected` state (ADR 0011 decision 8, L10.1) contributes its last known listing marked `stale` and is never waited for. The model call has the platform's `completion_timeout_ms`, after which the grammar ranking is shown. A tab on an LTE link behaves like a tab anywhere.

8. **Suggestions are not audited as actions, and refused proposals are counted.** The audit stream (ADR 0005) records what ran. A suggestion that was not accepted is not an event. The session keeps counters under `/sys/session/<id>/completion/`: calls, cache hits, proposals dropped for being outside the set, proposals dropped for failing type check, destructive verbs proposed. The platform reads the counters to see what the model keeps proposing; nothing else reads them.

9. **Scripts never depend on completion.** Completion exists only at an interactive prompt. A script, a `cron` line, a `batch` file or a unit definition is parsed as written; a model is never consulted when there is no cursor. Translation is never implicit: a comment in a script is a comment.

10. **Nothing is executed to compute a completion.** Completion reads metadata, the annotation database and `explain`, and `explain` writes nothing (ADR 0013 decision 3). A candidate's gloss may say "would create an Indexed Job of 48 partitions, $3 to $5"; it is computed from the plan, not from a trial run.

## Declined

| Proposal | Why not | What replaces it |
|---|---|---|
| an `ask` or `ai` verb that runs what the model decides | the model would become an actor with the user's identity and no plan gate | translation into a draft line that the user runs through `explain` |
| natural language as the primary syntax | the typed grammar is the semantic support; a model on top of it ranks, a model instead of it guesses | decision 1 and 2 |
| the model fills defaults the user left out | hidden defaults are misroute S3 and the platform's to set (ADR 0010) | defaults come from the profile and `explain` prints them |
| a model key in `~/.convoyrc` or an environment variable | a value in the client or the environment is a leak (ADR 0007) | the model is a mounted device with an identity option |
| fleet-wide completion context | a cross-tenant view the user does not have | identity-filtered candidates per mount |
| completion that lists a bucket or a topic live | priced and slow, and it hangs on a disconnected site | the metadata cache with age shown |

## Consequences

- `/dev/model` is a tenth device class and belongs in ADR 0011 decision 9's list; `caps` reports `model:<provider>` drivers like `queue:kafka` and `fs:hdfs`.
- `explain --json` becomes load-bearing: the ranker reads it, so its schema (open in ADR 0013) is needed before any model is attached.
- ROADMAP M2 gains type-directed completion without a model, since it needs only the annotation database and the verb tables; the model device belongs with the other device classes in M5 and M8.
- [US-14](../user-stories/US-14-assisted-shell.md) probes this record in the regulated setting of US-08 and the edge setting of US-10 under the AI Engineer and Security personas. It raised six findings: L14.2 (the translation comment is free text and needs the artifact scrub) and L14.4 (provider-side retention as a `/sys` file and a `retention=0` mount option) are applied inline in decisions 5 and 3 as of 2026-11-09; L14.1, L14.3 and L14.5 are rules on the fallback notice, the number of glossed candidates and the identity and residency in force at a keystroke; L14.6 accepts that ranking is non-deterministic. They cluster with misroute S13 in the register: context that leaves the session is data at rest somewhere else.
- Open: whether a gloss may quote a receipt of the user's own earlier run; what the ranker is allowed to learn across sessions, if anything; the `scope` option's default.

## Rules · 2026-11-09 · from the user-story probes

Each line is a `rule` finding from `docs/user-stories/README.md` written here as decided; the story holds the reasoning.

- **L14.1** (US-14) every list states its source, `ranked by <device>` or `grammar order`, and the reason for the latter (timeout, cap, residency, stale listing); the notice is the same line that appears in `/sys/cost/session/completion` counters.
- **L14.5** (US-14) candidates are filtered by the effective identity at the keystroke and re-filtered when it changes; the residency of outbound context is the intersection of the model mount's `residency` and the topology of every device the line names, and an empty intersection means the model is not consulted, with the notice of L14.1.
