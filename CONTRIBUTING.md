# Contributing

Convoy is in the design phase. Contributions are documents, decisions and reviews, not code, until the first milestone in `ROADMAP.md` opens.

## How to propose a change

1. **Small corrections** (a wrong Kubernetes fact, a broken example): open a pull request against the document.
2. **A new decision or a reversal:** add a record under `docs/adr/` using the existing format (Context, Decision, Consequences, Status). Link the documents it changes. Open a pull request.
3. **A new user story or abstraction:** add it to `docs/stories.md` in the existing shape: command, what it maps to, and what happens on failure. A story without failure behaviour is not complete.

## Review rules

- **Name the version.** Any claim about what Kubernetes, Argo, Kueue or a CSI driver does must say which version it holds for. "Beta since 1.30" is a claim; "supported" is not.
- **Prior art first.** If PaSh, Slurm, Dagger, Plan 9 or another system already solved the problem, cite it and say what differs.
- **Emulation, not illusion.** A proposal that hides a semantic difference behind a heuristic will be asked to surface it instead.
- **No numbers you do not have.** Designs that branch on output size, runtime duration or other quantities unknown at plan time will be asked to use a quantity that is known.

## Style

Short sentences. Tables for parallel items. Every example that is YAML must parse. Every command example must be one a user could type.
