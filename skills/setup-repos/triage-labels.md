# Triage Labels

The skills speak in terms of canonical role names. This file maps those roles to the actual label strings used in this repo's issue tracker. Edit the middle column when this repo already uses different names — `triage` will then apply the existing labels instead of creating duplicates.

Labels come in two axes. A triaged issue carries exactly one from each.

## State

| Canonical role    | Label in this repo | Meaning                                                |
| ----------------- | ------------------ | ------------------------------------------------------ |
| `needs-triage`    | `needs-triage`     | Filed, not yet assessed                                |
| `needs-info`      | `needs-info`       | Blocked on an answer from a human                      |
| `ready-for-agent` | `ready-for-agent`  | Specified well enough for an agent to pick up unattended |
| `ready-for-human` | `ready-for-human`  | Needs a human; an agent should not grab it             |
| `wontfix`         | `wontfix`          | Closed by decision, not by work                        |

## Category

| Canonical role | Label in this repo | Meaning                       |
| -------------- | ------------------ | ----------------------------- |
| `bug`          | `bug`              | Something is broken           |
| `enhancement`  | `enhancement`      | New feature or improvement    |

On GitHub, `bug` and `enhancement` ship with every repo. On GitLab neither exists until created — see the bootstrap block in `docs/agents/issue-tracker.md`.

When a skill names a role (e.g. "apply the AFK-ready triage label"), use the corresponding string from the middle column.
