---
name: pr
description: Push the current branch and create or update its PR. Invoke when the user runs /pr or asks to ship, push, open a PR, submit a pull request, or send for review — even if they don't say "PR" explicitly.
---

# PR

## Input

`$ARGUMENTS` — optional. `only` = skip the auto-review in step 6. Anything else is passed through to `/flow:review-pr` as effort level and/or review focus. Always operates on the current branch.

## Context-aware shortcuts

- Skip `git status -sb` / `git branch --show-current` if you ran them this session and nothing changed.
- Skip `gh pr view` if a PR was created or fetched earlier this session — use that number.

## Steps

1. **Pre-checks**:
   - `git status -sb`. Uncommitted changes → invoke the `/flow:commit` skill to stage and commit them, then continue. `/flow:commit` owns staging, scope, the comment audit, and branch create/rename — never inline a bare `git add -A` commit here.
   - After committing: on `dev`/`main`/`master` or no commits ahead of `dev` → abort.

2. **Sync with dev**: `git pull --rebase origin dev`. Real conflicts → abort and report.

3. **Push**: `git push -u origin HEAD` (`--force-with-lease` if rebased).

4. **Create or update PR**:
   - PR number known from context → `gh pr edit <n> --title "..." --body "..."`. On "no PR" error, fall through to create.
   - Otherwise → `gh pr view --json number,url`. Exit 0 = exists, edit it. Non-zero = none, create:
     `gh pr create --base dev --title "..." --body "..."`

5. **Return** the PR URL.

6. **Review**: if `$ARGUMENTS` is `only`, skip. Otherwise invoke `/flow:review-pr` with the PR number from step 4, appending `$ARGUMENTS` if present.

## Title and body

**Title:** conventional commit format, scope = the area the change touches.

**Body:** free-form — a short explanation of what changed and why, at whatever depth the change warrants (prose or bullets; your call).

## Rules

- One PR = one concern.
- No `Generated with Claude Code` footer.
- Cross-repo references: bare `#NNN` auto-links to the current repo. Use `owner/repo#NNN` when pointing to another repo.
