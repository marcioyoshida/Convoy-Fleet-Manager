# ADR 0005: The session is a server-side object with a thin client

**Status:** Proposed · 2026-11-08
**Supersedes / relates to:** docs/stories.md (abstractions); docs/debate-review.md §7

## Context

Earlier drafts made the session a local process and ran the fastest backend inside it. A laptop sleeping would kill every local stage; there would be no nohup, screen or tmux; the audit stream would be client-controlled.

## Decision

One session pod per user runs the parser, planner, local backend, mount table, env table and audit writer. The terminal attaches over WebSocket. `nohup` on a local command promotes it to a Job.

## Consequences

Detach and reattach work. The audit stream is trustworthy. Session state (mount table, env, cwd) is server-side and inherited by launched processes at creation time, Plan 9 style. Identity, ownership and networking all store their state here, so this decision precedes them.

## Rules · 2026-11-09 · from the user-story probes

Each line is a `rule` finding from `docs/user-stories/README.md` written here as decided; the story holds the reasoning.

- **L6.7** (US-06) prompts in non-interactive sessions are exit 3 with the required flag (`--yes`), consistent with `rm -i`.
