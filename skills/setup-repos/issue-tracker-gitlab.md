# Issue tracker: GitLab

Issues and PRDs for this repo live as GitLab issues. Use the [`glab`](https://gitlab.com/gitlab-org/cli) CLI for all operations.

## Conventions

- **Create an issue**: write the description to a file first, then `glab issue create --title "..." --description "$(cat <path>)" --label "..."`. Passing the file's contents as one quoted argument avoids shell-mangling the markdown; an unquoted heredoc breaks on backticks and `$`.
- **Read an issue**: `glab issue view <number> --comments`. Use `-F json` for machine-readable output.
- **List issues**: `glab issue list -F json` with appropriate `--label` filters.
- **Comment on an issue**: `glab issue note <number> --message "..."`. GitLab calls comments "notes".
- **Apply / remove labels**: `glab issue update <number> --label "..."` / `--unlabel "..."`. Multiple labels can be comma-separated or by repeating the flag.
- **Close**: `glab issue close <number>`. `glab issue close` does not accept a closing comment, so post the explanation first with `glab issue note <number> --message "..."`, then close.
- **Merge requests**: GitLab calls PRs "merge requests". Use `glab mr create`, `glab mr view`, `glab mr note`, etc. — the same shape as `gh pr ...` with `mr` in place of `pr` and `note`/`--message` in place of `comment`/`--body`.

Infer the repo from `git remote -v` — `glab` does this automatically when run inside a clone.

## Merge requests as a triage surface

**MRs as a request surface: no.** _(Set to `yes` if this repo treats external merge requests as feature requests; `/triage` reads this flag.)_

When set to `yes`, MRs run through the same labels and states as issues, using the `glab mr` equivalents:

- **Read an MR**: `glab mr view <number> --comments` and `glab mr diff <number>` for the diff.
- **List external MRs for triage**: `glab mr list -F json`, then keep only MRs whose author is not a project member/owner (a contributor's MR, not a maintainer's in-flight work).
- **Comment / label / close**: `glab mr note`, `glab mr update --label`/`--unlabel`, `glab mr close`.

Unlike GitHub, GitLab numbers issues and MRs separately, so `#42` is unambiguous once you know which surface the maintainer means.

## When a skill says "publish to the issue tracker"

Create a GitLab issue.

## When a skill says "fetch the relevant ticket"

Run `glab issue view <number> --comments`.

## Blocking edges

GitLab has a **native blocking link**, but it is a Premium/Ultimate feature. Where it's available, create blockers first so their numbers exist, then post the `/blocked_by` quick action as a note on the blocked issue:

```sh
glab issue note <blocked> --message "/blocked_by #<blocker>"
```

Read the links back with `glab api projects/:id/issues/:iid/links`.

On the free tier (or wherever the feature isn't available), fall back to a line at the top of the blocked issue's description:

```
Blocked by: #12, #13
```

A ticket is unblocked when every issue it lists is closed.

## Bootstrapping the labels in this repo

GitLab ships no triage labels by default — create the five states once, or the first `glab issue update --label` fails:

```sh
glab label create --name "needs-triage"    --description "Filed, not yet assessed"                       --color "#FBCA04"
glab label create --name "needs-info"      --description "Blocked on an answer from a human"             --color "#D93F0B"
glab label create --name "ready-for-agent" --description "Specified well enough for an agent to pick up" --color "#0E8A16"
glab label create --name "ready-for-human" --description "Needs a human; an agent should not grab it"    --color "#1D76DB"
glab label create --name "wontfix"         --description "Closed by decision, not by work"               --color "#FFFFFF"
```

Also create `bug` and `enhancement` if this repo's triage uses the category labels — unlike GitHub, GitLab doesn't provide them. If this repo overrides the default label strings, use the strings from `docs/agents/triage-labels.md` instead of the names above.
