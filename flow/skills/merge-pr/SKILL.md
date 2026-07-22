---
name: merge-pr
description: Merge a pull request immediately and detach the local worktree. Invoke when the user runs /merge-pr or asks to merge, land, ship, or close out a PR.
---

# Merge PR

## Input

$ARGUMENTS — optional PR number. If omitted, use the current branch's open PR.

## Steps

1. Resolve PR number: argument → use it; else `gh pr view --json number`; else ask. Skip if already known this session.

2. Capture the branch: `gh pr view <number> --json headRefName -q .headRefName`.

3. Merge: `gh pr merge <number> --squash`.

4. Close linked issue if this PR clears the last open item of a linked issue (`#N` / `Part of #N`): `gh issue close <N> --comment "<merged PRs>"`. `gh issue view <N>` only when unsure.

5. Cleanup, in order:
   - `git fetch origin dev && git checkout --detach origin/dev`
   - `git push origin --delete <branch>`
   - `git branch -D <branch>`
