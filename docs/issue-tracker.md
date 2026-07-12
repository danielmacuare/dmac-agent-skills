# Issue tracker

Global default config for the skills that read or publish issues (`code-review`, `to-spec`, `to-tickets`).

A project overrides this file by providing its own `docs/agents/issue-tracker.md`. Skills check the project file first and fall back to this one.

## Tracker

GitHub, via the `gh` CLI. The repo is whatever `gh repo view --json nameWithOwner` resolves from the current `git remote`.

If there is no GitHub remote, there is no tracker: write specs and tickets as local files under `.scratch/` instead, and skip everything below.

## Fetching an issue

```sh
gh issue view <number> --json number,title,body,labels,state,comments
gh pr view <number> --json number,title,body,labels,state,comments   # for !67 / PR-shaped refs
```

## Creating an issue

Write the body to a file first — `--body-file` avoids shell-quoting the markdown.

```sh
gh issue create --title "<title>" --body-file <path> --label "<label>"
```

`gh issue create` prints the new issue's URL; parse the number from it when later issues need to reference this one.

## Blocking edges

GitHub has no first-class "blocked by" field on plain issues. Create blockers first so their numbers exist, then reference them from the blocked issue's body:

```
Blocked by: #12, #13
```

Where a repo already uses GitHub sub-issues, attach the child to its parent instead:

```sh
gh api -X POST repos/{owner}/{repo}/issues/<parent>/sub_issues -f sub_issue_id=<child-id>
```

(`sub_issue_id` is the issue's **id**, not its number: `gh issue view <n> --json id`.)

## Triage labels

The label strings are identical to the canonical role names the `triage` skill uses — no mapping needed.

**State** (exactly one per issue):

| Label | Meaning |
|---|---|
| `needs-triage` | Filed, not yet assessed. |
| `needs-info` | Blocked on an answer from a human. |
| `ready-for-agent` | Specified well enough for an agent to pick up unattended. |
| `ready-for-human` | Needs a human; an agent should not grab it. |
| `wontfix` | Closed by decision, not by work. |

**Category** (exactly one per triaged issue) — GitHub's defaults, already present in most repos:

| Label | Meaning |
|---|---|
| `bug` | Something is broken. |
| `enhancement` | New feature or improvement. |

## Bootstrapping the labels in a repo

`bug` and `enhancement` ship with every GitHub repo. The five state labels do not:

```sh
gh label create needs-triage    --description "Filed, not yet assessed"                        --color FBCA04
gh label create needs-info      --description "Blocked on an answer from a human"              --color D93F0B
gh label create ready-for-agent --description "Specified well enough for an agent to pick up"  --color 0E8A16
gh label create ready-for-human --description "Needs a human; an agent should not grab it"     --color 1D76DB
gh label create wontfix         --description "Closed by decision, not by work"                --color FFFFFF
```

`gh label create` fails if the label exists; add `--force` to overwrite.
