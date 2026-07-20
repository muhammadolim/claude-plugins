# muhammadolim/claude-plugins

Personal Claude Code plugin marketplace, usable in any project.

## Plugins

### `flow`

Repo-agnostic git workflow skills:

| Skill | Command | Does |
|---|---|---|
| merge-pr | `/flow:merge-pr` | Squash-merge and detach the worktree |
| issue | `/flow:issue` | File a GitHub issue in the current repo (or a mapped sibling repo) |
| release | `/flow:release` | Draft/update the release PR (dev → main/master), grouped changelog, flags migrations |

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
