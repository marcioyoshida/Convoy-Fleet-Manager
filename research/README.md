# Research notes

The files under `notes/` are AI-generated takes on how a POSIX shell could map onto Kubernetes, collected between 2026-11-07 and 2026-11-08 while the Convoy design was being debated. They are kept verbatim, converted from PDF and DOCX to text, because the design documents argue against specific claims in them and the argument should stay checkable.

They are **not** design documents. Several contain errors that `docs/debate-review.md` and `docs/pipeline-compiler.md` identify and correct. Read them as the position being reviewed.

| Note | Reviewed in |
|---|---|
| `what-are-the-capabilities-of-this.md` (59 pages: capability tables, translator API, POSIX gaps, user stories, position paper) | `docs/debate-review.md` and the story critique in `docs/stories.md` |
| `kubernetes-based-os.md` (the position paper on its own) | same |
| `how-would-unix-style-pipes-compile-into-distribut.md` | superseded by `docs/pipeline-compiler.md` |
| `how-would-disk-operations-and-filesystem-mounts-m.md` | `docs/debate-review.md`, verbs section |
| `umount-maps-to-pvc-deletion-unmount-detaches-it.md` (a reply to the disk review) | `docs/debate-review.md` |
| the nine `how-are…` / `how-do…` / `how-could…` / `package-management…` notes | `docs/debate-review.md`, one section each |

The original binaries are kept locally under `sources/` and are excluded from the repository.
