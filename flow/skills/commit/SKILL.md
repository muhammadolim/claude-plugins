---
name: commit
description: Stage and commit changes, creating or renaming the branch as needed. Invoke when the user runs /commit or asks to commit, save this work, check in changes, snapshot progress, or finalize their edits — even if they don't say "commit" explicitly.
---

# Commit

## Steps

1. **Pre-checks** (all parallel):
   - `git status` (no `-uall`)
   - `git log --oneline dev..HEAD`
   - `git branch --show-current`

2. **Fix state if needed**:
   - Detached HEAD or no branch → `git checkout -b feat/<scope>-<description>`
   - No unstaged/uncommitted changes and no commits ahead of dev → abort, nothing to commit

3. **Audit new/changed comments** in the diff:
   - Drop comments that narrate WHAT the code does or restate the obvious.
   - Drop comments referencing the current task, PR, or fix.
   - Keep only WHY: non-obvious constraints, tradeoffs, workarounds, hidden invariants.
   - One line when it fits.
   - Edit the file before staging.

4. **Commit** (skip if nothing to commit):

   Decide shape from conversation context:
   - **Amend** previous commit (`git commit --amend --no-edit`, or `--amend` to edit the message) when it was made this session, isn't pushed yet, and the new changes fix or polish it (typo, lint, missed file, follow-up tweak in the same area).
   - **Split into multiple commits** when changes span clearly distinct concerns (different areas, unrelated fix mixed with feature work). Stage each group separately and commit one after the other.
   - **Hunk-level staging** (`git add -p`) when one file mixes in-scope and out-of-scope hunks.
   - Otherwise, single commit.

   Common rules:
   - Stage relevant files — never `git add .` or `git add -A`
   - Conventional commit format
   - Scope = the area of the codebase the change touches

5. **Rename branch when scope grew**:
   - Look at all commits in `git log --oneline dev..HEAD`. If they span more than the branch name's original scope, rename to cover the actual work. When uncertain, prefer a broader name over a misleading narrow one.
   - Apply: `git branch -m <old> <new>`, then `git push origin --delete <old>` if old name was pushed.
