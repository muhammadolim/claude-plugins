---
name: release
description: Create a release PR from the integration branch (dev) to the repo's production branch (main or master) with a grouped, user-facing changelog. Invoke when the user runs /release or asks to cut, ship, or draft a release, or to deploy/promote to production. Auto-detects the production branch and flags any migration/backfill/seed scripts in the release range.
---

# Release PR

Flow releases `dev` → the repo's **production branch**. That branch differs by repo (`main`, `master`, …), so detect it rather than assume.

## Steps

1. **Detect the production branch** — resolve once, call it `<base>`:
   `gh repo view --json defaultBranchRef --jq .defaultBranchRef.name` (typically `main` or `master`). If the default branch is `dev`, ask the user which branch to release into.

2. **Fetch & diff** — run the fetches in parallel, then log:
   - `git fetch origin <base>` + `git fetch origin dev` (parallel)
   - `git log --oneline origin/<base>..origin/dev`
   - `git diff --name-only origin/<base>..origin/dev | grep -iE 'migrat|backfill|seed'` — flag migration scripts

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
- If the diff includes migration/backfill/seed scripts, append a `## ⚠️ Migrations` section after the summary: list each script with its run command; note they run manually against production, not on deploy
- Descriptions are user-facing — say what changed for the user, not how. No internal names (PATCH, slice, slug, optimistic, ref, hook), no file/type names, no implementation mechanics. One short phrase per change, comma-separated within an area
- No PR number references (no `(#123)`, no links) in the body — description only
- No test plan, no `Generated with Claude Code` footer
- Don't merge — leave the PR open for owner review
