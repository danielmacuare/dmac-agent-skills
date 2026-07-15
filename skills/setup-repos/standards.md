# Coding Standards

Which standards apply to this repo, and where they live. `implement`, `tdd`, and
`code-review` read this file.

## Languages

This repo is written in: **[languages detected — e.g. Python, Terraform]**

## Sources, in descending precedence

1. **This repo** — [name any repo-local standards file, e.g. `CODING_STANDARDS.md` or
   `CONTRIBUTING.md`. Write "none" if the repo documents nothing.]
2. **Global** — for each language above, `~/.agents/docs/ai/<lang>/coding-standards.md`.
   Follow that file's Categories index and load only the numbered files matching the task
   at hand, not the whole tree.
3. **Smell baseline** — the Fowler code smells carried by `code-review` itself.

A nearer tier overrides a further one. Where this repo documents a rule that contradicts the
global standards, this repo wins — say so explicitly in tier 1 rather than editing the
global tree.

## Repo-specific overrides

[Anything this repo does differently from the global standards. One bullet per rule, with
the reason. Delete this section if there's nothing to say.]
