# Issue tracker: GitHub

Issues and PRDs for this repo live as GitHub issues. Use the `gh` CLI for all operations.

## Conventions

- **Create an issue**: write the body to a file first, then `gh issue create --title "..." --body-file <path> --label "..."`. `--body-file` avoids shell-quoting the markdown; a `--body "..."` heredoc mangles backticks and `$`. `gh issue create` prints the new issue's URL — parse the number from it when later issues need to reference this one.
- **Read an issue**: `gh issue view <number> --comments`, filtering comments by `jq` and also fetching labels.
- **List issues**: `gh issue list --state open --json number,title,body,labels,comments --jq '[.[] | {number, title, body, labels: [.labels[].name], comments: [.comments[].body]}]'` with appropriate `--label` and `--state` filters.
- **Comment on an issue**: `gh issue comment <number> --body "..."`
- **Apply / remove labels**: `gh issue edit <number> --add-label "..."` / `--remove-label "..."`
- **Close**: `gh issue close <number> --comment "..."`

Infer the repo from `git remote -v` — `gh` does this automatically when run inside a clone.

## Pull requests as a triage surface

**PRs as a request surface: no.** _(Set to `yes` if this repo treats external PRs as feature requests; `/triage` reads this flag.)_

When set to `yes`, PRs run through the same labels and states as issues, using the `gh pr` equivalents:

- **Read a PR**: `gh pr view <number> --comments` and `gh pr diff <number>` for the diff.
- **List external PRs for triage**: `gh pr list --state open --json number,title,body,labels,author,authorAssociation,comments` then keep only `authorAssociation` of `CONTRIBUTOR`, `FIRST_TIME_CONTRIBUTOR`, or `NONE` (drop `OWNER`/`MEMBER`/`COLLABORATOR`).
- **Comment / label / close**: `gh pr comment`, `gh pr edit --add-label`/`--remove-label`, `gh pr close`.

GitHub shares one number space across issues and PRs, so a bare `#42` may be either — resolve with `gh pr view 42` and fall back to `gh issue view 42`.

## When a skill says "publish to the issue tracker"

Create a GitHub issue.

## When a skill says "fetch the relevant ticket"

Run `gh issue view <number> --comments`.

## Blocking edges

GitHub has no first-class "blocked by" field on plain issues. Create blockers first so their numbers exist, then reference them from the blocked issue's body:

```
Blocked by: #12, #13
```

Where a repo already uses GitHub sub-issues, attach the child to its parent instead:

```sh
gh api -X POST repos/{owner}/{repo}/issues/<parent>/sub_issues -f sub_issue_id=<child-id>
```

`sub_issue_id` is the issue's **id**, not its number: `gh issue view <n> --json id`.

A ticket is unblocked when every issue it lists is closed.

## Bootstrapping the labels in this repo

`bug` and `enhancement` ship with every GitHub repo. The five triage state labels do not — create them once, or the first `gh issue edit --add-label` fails with "label not found":

```sh
gh label create needs-triage    --description "Filed, not yet assessed"                        --color FBCA04
gh label create needs-info      --description "Blocked on an answer from a human"              --color D93F0B
gh label create ready-for-agent --description "Specified well enough for an agent to pick up"  --color 0E8A16
gh label create ready-for-human --description "Needs a human; an agent should not grab it"     --color 1D76DB
gh label create wontfix         --description "Closed by decision, not by work"                --color FFFFFF
```

`gh label create` fails if the label already exists; add `--force` to overwrite. If this repo overrides the default label strings, use the strings from `docs/agents/triage-labels.md` instead of the names above.
