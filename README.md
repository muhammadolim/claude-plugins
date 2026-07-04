# muhammadolim/claude-plugins

Personal Claude Code plugin marketplace, usable in any project.

## Plugins

### `flow`

Repo-agnostic git workflow skills:

| Skill | Command | Does |
|---|---|---|
| commit | `/flow:commit` | Stage and commit, creating/renaming the branch |
| pr | `/flow:pr` | Push and create/update the PR, then review |
| review-pr | `/flow:review-pr` | Find issues (via `code-review`), fix, verify, comment |
| merge-pr | `/flow:merge-pr` | Squash-merge and detach the worktree |
| issue | `/flow:issue` | File a GitHub issue in the current repo (or a mapped sibling repo) |
| release | `/flow:release` | Draft/update the release PR (dev → main/master), grouped changelog, flags migrations |

`review-pr` reads the PR verify command from the repo's `CLAUDE.md` / `CLAUDE.local.md` (falls back to detecting `typecheck` → `build` → `tsc --noEmit`), so it adapts per project.

`issue` defaults to the current repo; cross-repo routing and assignee aliases come from the repo's `CLAUDE.md` / `CLAUDE.local.md`, so it stays team-neutral.

## Install

```
/plugin marketplace add muhammadolim/claude-plugins
/plugin install flow@mo
```

## Update

```
/plugin marketplace update
/plugin update flow
```

## Local development

```
claude --plugin-dir ./flow
```
