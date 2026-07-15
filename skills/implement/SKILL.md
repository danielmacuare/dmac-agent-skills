---
name: implement
description: "Implement a piece of work based on a spec or set of tickets."
disable-model-invocation: true
---

Implement the work described by the user in the spec or tickets.

Issue-tracker workflow: read `docs/agents/issue-tracker.md` in the current project. If it doesn't exist, the project hasn't been set up — run `/setup-repos` first.

Domain doc conventions: read `docs/agents/domain.md` in the current project if it exists.

Coding standards: if the repo documents its own (`CODING_STANDARDS.md`, `CONTRIBUTING.md`, or whatever `docs/agents/standards.md` names), those win. Otherwise, for each language you touch, read `~/.agents/docs/ai/<lang>/coding-standards.md` if it exists — follow its Categories index and load only the numbered files matching your current task, not the whole tree.

Use /tdd where possible, at pre-agreed seams.

Run typechecking regularly, single test files regularly, and the full test suite once at the end.

Once done, use /code-review to review the work.

Commit your work to the current branch.
