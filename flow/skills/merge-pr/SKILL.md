---
name: merge-pr
description: Merge a pull request immediately and detach the local worktree. Invoke when the user runs /merge-pr or asks to merge, land, ship, or close out a PR.
---

# Merge PR

## Input

$ARGUMENTS — optional PR number. If omitted, use the current branch's open PR.

## Context-aware shortcut

If the PR number is already known from earlier this session, skip step 1.

## Steps

1. **Resolve PR number**:
   - Argument given → use it
   - No argument → `gh pr view --json number` for current branch
   - Still none → ask the user

2. **Merge**: `gh pr merge <number> --squash --delete-branch`

3. **Close linked issue if done**: when this PR clears the last open item of a linked issue (`#N` / `Part of #N`), close it: `gh issue close <N> --comment "<merged PRs>"`. Judge "done" from session context; `gh issue view <N>` only when unsure.

4. **Cleanup**: `git fetch origin dev && git checkout --detach origin/dev`
