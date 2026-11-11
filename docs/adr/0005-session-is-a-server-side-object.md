# ADR 0005: The session is a server-side object with a thin client

**Status:** Proposed · 2026-11-08
**Supersedes / relates to:** docs/stories.md (abstractions); docs/debate-review.md §7

## Context

Earlier drafts made the session a local process and ran the fastest backend inside it. A laptop sleeping would kill every local stage; there would be no nohup, screen or tmux; the audit stream would be client-controlled.

## Decision

One session pod per user runs the parser, planner, local backend, mount table, env table and audit writer. The terminal attaches over WebSocket. `nohup` on a local command promotes it to a Job.

## Consequences

Detach and reattach work. The audit stream is trustworthy. Session state (mount table, env, cwd) is server-side and inherited by launched processes at creation time, Plan 9 style. Identity, ownership and networking all store their state here, so this decision precedes them. Identity is drafted in ADR 0016: the pod's ServiceAccount is only the runtime handle, the session's identity is the user and groups of the login token. The design of the session itself (placement per residency, the journal and restore, the attach protocol, a shell per terminal, idle and upgrade, the prompt) is drafted in [ADR 0020](0020-a-session-is-a-journal-not-a-pod.md).

Note · 2026-11-10: units are the exception to inheritance. A unit outlives the session and every release re-creates its pods, so [ADR 0013](0013-native-verbs-have-one-shape.md) decision 4 (revised with L24.3) makes a unit's environment its declaration only; a session variable reaches a unit only when named with `--env NAME`. Every other launched process inherits as above.

## Rules · 2026-11-09 · from the user-story probes

Each line is a `rule` finding from `docs/user-stories/README.md` written here as decided; the story holds the reasoning.

- **L6.7** (US-06) prompts in non-interactive sessions are exit 3 with the required flag (`--yes`), consistent with `rm -i`.
- **L20.9** (US-20) `nohup` promotion re-plans reach: a unit the command reached through the session's hosts view must be a granted `--connect` on the Job, else promotion refuses naming it (ADR 0019 rule L20.9).
