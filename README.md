# .agents

Collection of Claude Code agent skills.

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

## Usage

Point Claude Code at this repo (or symlink `skills/`) so skills load automatically, or invoke a skill by name via the `Skill` tool / `/<skill-name>` slash command.
