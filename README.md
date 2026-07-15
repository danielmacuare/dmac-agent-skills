# .agents

Collection of Claude Code agent skills.

Some of these skills are heavily inspired by Matt Pocock skills: <https://github.com/mattpocock/skills/tree/main> and customised to my use case. Some other skills are completely my own.

## Skills

| Skill | Purpose |
|---|---|
| `code-review` | Two-axis review (standards + spec) of a diff since a fixed point |
| `find-skills` | Discover and install skills from the agent skills ecosystem |
| `frontend-design` | Build production-grade, distinctive frontend UI |
| `grill-me` | Relentless interview to sharpen a plan or design |
| `grill-with-docs` | Same interview, also produces ADRs and glossary docs |
| `skill-creator` | Guide/tooling for creating new skills |
| `tdd` | Test-driven development (red-green-refactor) reference |
| `teach` | Multi-session teaching of a topic to the user |
| `to-spec` | Turn conversation into a spec/PRD |
| `to-tickets` | Break a plan/spec into tracer-bullet tickets |
| `web-design-guidelines` | Audit UI against Web Interface Guidelines |
| `writing-great-skills` | Reference for writing skills well |

## Layout

Each skill lives in `skills/<name>/` with a `SKILL.md` (frontmatter: `name`, `description`, optional `disable-model-invocation`) plus any supporting scripts/docs/licenses.

## Config

Config is per-project, not global. Run `/setup-repos` in a repo to scaffold it:

- `docs/agents/issue-tracker.md` — which tracker to publish to and how to fetch, create, comment, and label issues. `code-review`, `to-spec`, and `to-tickets` read it.
- `docs/agents/triage-labels.md` — the label strings behind the five canonical triage roles. `triage` reads it.
- `docs/agents/domain.md` — where `CONTEXT.md` and ADRs live, and the rules for reading them.
- `docs/agents/standards.md` — the repo's languages and its standards precedence chain. `implement`, `tdd`, and `code-review` read it.

A repo without these files hasn't been set up; skills that need them will say so.

`docs/ai/<lang>/coding-standards.md` holds the global, per-language coding standards that apply across every repo — currently Python. A repo's own `CODING_STANDARDS.md` overrides them; the Fowler smell baseline in `code-review` backstops both.

## Usage

Point Claude Code at this repo (or symlink `skills/`) so skills load automatically, or invoke a skill by name via the `Skill` tool / `/<skill-name>` slash command.
