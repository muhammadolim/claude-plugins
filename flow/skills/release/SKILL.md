---
name: release
description: Create a release PR from the integration branch (dev) to the repo's production branch (main or master) with a grouped, user-facing changelog. Invoke when the user runs /release or asks to cut, ship, or draft a release, or to deploy/promote to production. Auto-detects the production branch and flags any data-migration scripts in the release range by inspecting the changed scripts, not just their names.
---

# Release PR

Flow releases `dev` → the repo's **production branch**. That branch differs by repo (`main`, `master`, …), so detect it rather than assume.

## Steps

1. **Detect the production branch** — resolve once, call it `<base>`:
   `gh repo view --json defaultBranchRef --jq .defaultBranchRef.name` (typically `main` or `master`). If the default branch is `dev`, ask the user which branch to release into.

2. **Fetch & diff** — run the fetches in parallel, then log:
   - `git fetch origin <base>` + `git fetch origin dev` (parallel)
   - `git log --oneline origin/<base>..origin/dev`
   - **Migrations — list files, don't name-grep.** One-off migrations live in `scripts/` (or `migrations/`) and are named for what they *do* (`drop-…`, `normalize-…`, `fix-…`, `reconcile-…`), so a name grep misses them. List every changed script, then open each new/changed one's docblock and decide whether it needs a manual prod run:
     - `git diff --name-only origin/<base>..origin/dev -- 'scripts/**' 'migrations/**' 'db/migrations/**'`
     - A name grep (`grep -iE 'migrat|backfill|seed|drop|normaliz|reconcile|dedupe'`) is a hint to skim first, never the gate — a script that matches nothing can still be a migration.

3. **Look for an existing release PR**:
   ```
   gh pr list --base <base> --head dev --state open --json number,url
   ```

4. **Create or update** — group the commits by feature area, then:
   - No PR → create:
     ```
     gh pr create --base <base> --head dev \
       --title "Release YYYY-MM-DD" \
       --body "$(cat <<'EOF'
     ## Summary
     - **Area:** description
     EOF
     )"
     ```
   - PR exists → regenerate the summary from the current diff and edit in place:
     ```
     gh pr edit <number> \
       --title "Release YYYY-MM-DD" \
       --body "$(cat <<'EOF'
     ## Summary
     - **Area:** description
     EOF
     )"
     ```

5. Return the PR URL.

## Rules

- Title: `Release YYYY-MM-DD` (today's date)
- Body: `## Summary` with bullets grouped by feature area using bold headers (e.g. `- **Meetings:** ...`). Combine related commits into one bullet per area
- If the release includes any data-migration script, append a `## ⚠️ Migrations` section after the summary: list each with its exact run command (read the command from the script's docblock, don't guess); note they run manually against production, not on deploy
- Never let the filename grep be the sole gate for migrations — every changed file under `scripts/`/`migrations/` is a candidate; read its docblock and classify before finalizing the body
- Descriptions are user-facing — say what changed for the user, not how. No internal names (PATCH, slice, slug, optimistic, ref, hook), no file/type names, no implementation mechanics. One short phrase per change, comma-separated within an area
- No PR number references (no `(#123)`, no links) in the body — description only
- No test plan, no `Generated with Claude Code` footer
- Don't merge — leave the PR open for owner review
