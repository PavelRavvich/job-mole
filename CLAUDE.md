# Coding conventions

1. Keep files to roughly 100–150 lines. A file growing past that is a
   signal it's doing too much — split it.
2. Type everything as fully as practical (function signatures, return
   types, data shapes) — no bare `dict`/`Any` where a concrete type fits.
3. Follow SRP and DRY at every level — functions, classes, and files each
   have one clear responsibility; don't duplicate logic across them.
4. Don't reach for OOP where a plain function solves it. Classes are for
   real state/identity, not a wrapper around one procedure.
5. Choose variable/function/class/file names carefully and in full — never
   abbreviate to save keystrokes.
6. No comments that restate what the code already says. Code should be
   self-documenting through naming and structure; a comment only earns its
   place by explaining a non-obvious *why*.
7. Cover everything reasonable with unit tests, especially when fixing a
   defect — a bug fix ships with a regression test.
8. Group packages by meaning/domain, not by type — no dumping-ground
   modules (e.g. no generic `utils/` catch-all).

See `docs/superpowers/specs/2026-09-22-job-match-agent-design.md` for the
project's architecture and design decisions.
